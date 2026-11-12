# AGENTS.md

This file provides guidance to CodeBuddy Code when working with code in this repository.

## Overview

x-ui is a single-binary web panel for managing the Xray proxy core. The Go process serves a Gin HTTP panel and supervises an `xray` subprocess, generating its config from database-backed inbounds. The frontend is server-rendered Go templates plus vendored Vue 2 / ant-design-vue assets — there is **no npm or frontend build step**.

Module name is `x-ui`, so internal imports are `x-ui/web/...`, `x-ui/xray`, etc.

## Build and run

Requires Go 1.16+ (CI uses 1.18) and **CGO enabled** because the SQLite driver uses cgo.

```bash
CGO_ENABLED=1 go build -o x-ui main.go   # same command as Dockerfile/CI
go run main.go                            # starts the panel (default port 54321)
go run main.go -v                         # print version from config/version
./x-ui                                    # no args also starts the panel
```

Other CLI subcommands (see main.go):

```bash
./x-ui setting -show true
./x-ui setting -port 54321 -username admin -password admin
./x-ui setting -reset
./x-ui v2-ui -db /etc/v2-ui/v2-ui.db     # migrate inbounds from v2-ui
```

Development mode:

```bash
XUI_DEBUG=true XUI_LOG_LEVEL=debug go run main.go
```

- `XUI_DEBUG=true` serves `web/html` and `web/assets` from disk, enables Gin debug mode and GORM SQL logging.
- Run from the repository root: templates/assets and `bin/` are resolved relative to the working directory.
- Without debug mode, templates/assets are read from the compiled-in `embed.FS`, so frontend edits require a rebuild.
- There is no test suite or linter configuration in the repo. Use `CGO_ENABLED=1 go build ./...` and `go vet ./...` as the available checks.

Docker: `docker build -t x-ui .` (multi-stage, builds with `go build main.go`).

Releases: pushing a tag matching `0.*` triggers `.github/workflows/release.yml`, which builds linux amd64/arm64/s390x with `CGO_ENABLED=1` and bundles the matching Xray binary from GitHub releases.

## Runtime layout and paths

- Database: `/etc/x-ui/x-ui.db` (derived from `config/name`, see config/config.go:48). GORM auto-migrates `User`, `Inbound`, `Setting` on startup; first run creates `admin/admin`.
- Settings live in the `settings` key/value table, not in files. Missing keys fall back to `defaultValueMap` in web/service/setting.go:23 (default panel port 54321, base path `/`, timezone `Asia/Shanghai`, embedded Xray template).
- Xray binary: `bin/xray-<GOOS>-<GOARCH>`; generated config: `bin/config.json`; geo data: `bin/geoip.dat`, `bin/geosite.dat` (xray/process.go:30). Paths are relative to the process CWD, so the binary is normally run from `/usr/local/x-ui`.
- `x-ui.sh` / `install.sh` are deployment/service-management scripts (systemd, ACME cert issuance), not development tools.

## Architecture

Request path: `main.go` → `web.Server` → Gin engine → controllers → services → `database` / `xray`.

### web server (`web/web.go`)

- Embeds `web/assets/*`, `web/html/*`, `web/translation/*`.
- Cookie session store keyed by a random `secret` setting persisted in the DB.
- Configurable base path; all routes are mounted under it.
- Optional TLS if `webCertFile`/`webKeyFile` are set, wrapped by `web/network.AutoHttpsListener`.
- Owns the cron scheduler and all background jobs (web/web.go:282): traffic collection every 10s, Xray liveness check every 30s, inbound expiry/quota check every 30s, optional Telegram stats notification.
- `SIGHUP` stops and recreates the server in-process (used by “restart panel”); SIGTERM exits.

### Controllers (`web/controller`)

- `index.go`: login/logout. `xui.go`: page routes under `/xui` guarded by `checkLogin`.
- `inbound.go`: CRUD JSON API under `/xui/inbound`. Mutations call `XrayService.SetToNeedRestart()`; they do **not** restart Xray synchronously.
- `setting.go`: settings, credentials, panel restart. `server.go`: system status and Xray version/install.
- `util.go`: unified response envelope `{success, msg, obj}` via `jsonMsg`/`jsonObj`; `html()` injects `base_path`, `title`, `request_uri`.

### Services (`web/service`)

- `XrayService`: global Xray process singleton. `RestartXray(false)` compares the desired config with the running one and skips the restart if identical (web/service/xray.go:88). The restart flag is polled every 10s by `InboundController`, so inbound edits take effect within ~10s.
- `InboundService`: CRUD (unique port), traffic accumulation, and auto-disabling inbounds whose quota/expiry is exceeded.
- `SettingService`: generic key/value access with per-type getters; `GetAllSetting` reflects over struct tags to map DB keys to `entity.AllSetting` fields by `json` tag.
- `ServerService`: CPU/mem/disk/net/TCP metrics, Xray version list from GitHub, and Xray binary download/replacement.
- `UserService`: single admin user lookup/update.

### Xray integration (`xray`)

- `Process` renders `xray.Config` to `bin/config.json` and runs the binary, capturing stdout/stderr into a bounded queue for error display.
- The config template stored in settings is parsed into `xray.Config`; non-inbound sections use `json_util.RawMessage` so arbitrary Xray JSON passes through. DB inbounds are appended as `InboundConfig` entries.
- Traffic is read via Xray's gRPC `StatsService` on the inbound tagged `api` (a `dokodemo-door` listener in the template). `GetTraffic(true)` resets counters on read (xray/process.go:232).
- Inbound tags are generated as `inbound-<port>`; traffic rows are matched back by tag.

### Database (`database`)

- GORM + SQLite. `model.Inbound` stores Xray protocol config as raw JSON strings (`Settings`, `StreamSettings`, `Sniffing`). `ExpiryTime` is Unix milliseconds; `DisableInvalidInbounds` compares against `time.Now().Unix() * 1000`.
- Protocol constants are `vmess`, `vless`, `Dokodemo-door` (capital D), `http`, `trojan`, `shadowsocks`.

### Frontend

- Pages are Go templates under `web/html`; shared pieces live in `web/html/common` and `web/html/xui/component`. Each page is rendered by a controller `html()` call.
- Vue 2 + ant-design-vue, axios, clipboard, qrcode are vendored under `web/assets`; models in `web/assets/js/model`.
- i18n: TOML files in `web/translation`. The `i18n` template function parses `{{param}}` placeholders out of the key and requires the number of passed params to match exactly, so key/param counts must stay in sync across all three translation files.

## Common change recipes

- **New setting**: add the field with matching `json`/`form` tags to `entity.AllSetting`, add a default (if needed) to `defaultValueMap`, and wire a getter/setter in `SettingService`.
- **New page**: add `web/html/.../*.html`, register a route calling `html(c, ...)`, and link it from `common_sider.html`.
- **New background task**: implement a type with `Run()` and add it in `Server.startTask()` or a controller's `startTask()`.
- **Protocol UI**: protocol forms are under `web/html/xui/form/protocol`, transport forms under `web/html/xui/form/stream`.
