# Repo Profile: emobiq-client-backend

Last updated: 2026-05-07
Confidence: Medium

## 1. Purpose

Go (Gin) HTTP API for the Emobiq low-code client backend: project/app CRUD-style operations, media and SQLite-backed in-app database APIs, plugin discovery via an external service manager, and mobile/web **build orchestration** (tar/scp to compiler, callbacks, socket broadcast). It uses **MariaDB** (shared metadata) plus **per-application SQLite** under configurable project storage. User identity for Bearer-protected routes is resolved via the **main backend** OAuth introspection API.

## 2. Tech Stack

- Language: Go (`go.mod` declares `go 1.14`; project docs reference newer Go for local dev)
- Framework: Gin (`github.com/gin-gonic/gin`)
- Runtime: long-lived HTTP server (`backend/main.go`), optional Docker (`docker-compose.yml`, `backend/Dockerfile`)
- Package manager: Go modules (`backend/go.mod`, module path `bitbucket.org/orangekloud/emobiq-v5-backend`)
- Database: MariaDB/MySQL and PostgreSQL drivers via GORM; **MariaDB** used for app/user/build metadata; **SQLite** per app for pages, snippets, tables, etc. (`backend/app/model/model.go`, `backend/config/database.json.sample`)
- Other important dependencies: GORM, `gin-contrib/sessions`, Gorilla WebSocket (library present; see §4), `golang.org/x/crypto`

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `backend/main.go` | Gin app, CORS/session/recovery middleware, `/v1` routes, port from config |
| `backend/go.mod` | Module path and dependencies |
| `backend/app/route/setup.go` | **Canonical HTTP route map** and auth grouping |
| `backend/app/controller/` | HTTP handlers |
| `backend/app/service/` | Business logic (build, plugins, audits, etc.) |
| `backend/app/model/` | GORM models; MariaDB vs SQLite selection |
| `backend/app/middleware/` | `Authorization` Bearer validation, `API-Key`, CORS, sessions |
| `backend/app/helper/config.go` | Loads JSON from `<app-root>/config/*.json` |
| `backend/app/library/compiler/` | Build artifact prep, SCP, compiler and AI-backend HTTP calls |
| `backend/config/*.json.sample` | Committed templates; runtime `*.json` gitignored (`make config` copies a subset — see §8) |
| `Makefile` | `run` / `stop` / `deploy` via Docker Compose; `config` bootstraps some config files |
| `docker-compose.yml` | Builds `backend/Dockerfile`, exposes 8080, mounts `backend/config` and `backend/log` |
| `AGENTS.md` | Repo-level agent notes (stack, config boundaries) |
| `backend/README.md` | Local setup, config list, examples (incl. optional WebSocket wiring sample) |

## 4. Exposed Endpoints

All routes below are prefixed with `/v1` (see `backend/app/route/setup.go`). Response shape is typically `{ success, data, message }` (`backend/app/controller/controller.go`).

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/v1/app/:appid/` | App metadata | `backend/app/controller` (App) | Bearer |
| `POST` | `/v1/app/:appid/icon` | Upload app icon | same | Bearer |
| `PATCH` | `/v1/app/:appid` | Update app | same | Bearer |
| `GET` | `/v1/global/:appid/` | Global app data | Global controller | Bearer |
| `PATCH` | `/v1/global/:appid` | Update global | same | Bearer |
| `GET` | `/v1/media/:appid/directories` | Media directories | `backend/app/controller/media.go` | Bearer |
| `GET` | `/v1/media/:appid/files` | Media files | same | Bearer |
| `POST` | `/v1/media/:appid` | Upload media | same | Bearer |
| `DELETE` | `/v1/media/:appid` | Delete media | same | Bearer |
| `GET` | `/v1/database/tables/:appid/` | List tables | `backend/app/controller/database.go` | Bearer |
| `POST` | `/v1/database/tables/:appid` | Create table | same | Bearer |
| `PATCH` | `/v1/database/tables/:appid/:id` | Update table | same | Bearer |
| `DELETE` | `/v1/database/tables/:appid/:id` | Delete table | same | Bearer |
| `GET` | `/v1/database/tables/:appid/:id/fields` | List fields | same | Bearer |
| `POST` | `/v1/database/tables/:appid/:id/fields` | Create field | same | Bearer |
| `PATCH` | `/v1/database/tables/:appid/:id/fields/:fieldid` | Update field | same | Bearer |
| `DELETE` | `/v1/database/tables/:appid/:id/fields/:fieldid` | Delete field | same | Bearer |
| `GET` | `/v1/database/data/:appid/table/:tableid/` | Table row data | same | Bearer |
| `POST` | `/v1/database/data/:appid/table/:tableid` | Create row | same | Bearer |
| `PATCH` | `/v1/database/data/:appid/table/:tableid` | Update row(s) | same | Bearer |
| `DELETE` | `/v1/database/data/:appid/table/:tableid` | Delete row(s) | same | Bearer |
| `GET` | `/v1/applications/:appid` | Application info | `backend/app/controller/application.go` | Bearer |
| `PATCH` | `/v1/applications/:appid` | Update application | same | Bearer |
| `POST` | `/v1/applications/:appid/build` | Trigger build | same | Bearer |
| `PATCH` | `/v1/applications/:appid/lock/update` | Lock update | same | Bearer |
| `POST` | `/v1/applications/:appid/build/complete` | Build completion callback | same | API-Key (`auth.json`) |
| `POST` | `/v1/applications/build/clear` | Clear stuck build status | same | API-Key |
| `GET` | `/v1/logs/:appid` | Audit logs | `backend/app/controller/audit.go` | Bearer **or** API-Key |
| `POST` | `/v1/logs` | Create audit log | same | Bearer **or** API-Key |
| `GET` | `/v1/plugins/search` | Plugin search | `backend/app/controller/plugins.go` | Bearer |
| `GET` | `/v1/plugins/process/editor/data` | Plugin editor payloads | same | Bearer |
| `GET` | `/v1/style/:appid` | Custom styles | `backend/app/controller/custom_style.go` | Bearer |
| `GET` | `/v1/packages/:appid/` | List packages | `backend/app/controller/package.go` | Bearer |
| `POST` | `/v1/packages/:appid/files` | Add package file | same | Bearer |
| `POST` | `/v1/packages/:appid/links` | Add package link | same | Bearer |
| `PATCH` | `/v1/packages/:appid/order` | Package order | same | Bearer |
| `DELETE` | `/v1/packages/:appid/:packageid` | Delete package | same | Bearer |

**Note:** `backend/app/controller/socket.go` defines WebSocket-style handlers, but they are **not** registered in `backend/app/route/setup.go` or `backend/main.go`. `backend/README.md` shows optional manual registration. Treat WebSocket exposure as **NEEDS CONFIRMATION** for deployed builds unless a fork wires them in.

## 5. Outbound API Calls

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| Main backend (config `api.main_backend.api`) | `POST` | `{api}/oauth/tokens/introspect` | Validate Bearer token and load user | `backend/app/middleware/middleware.go` |
| Service manager (config `api.service_manager.api`) | `GET` | `{api}/search?q=...&user_id=...&company_id=...&platform=...&with_public=1` | Plugin search | `backend/app/service/plugin.go` |
| Service manager | `GET` | `{api}/package?id=...` | Plugin package metadata | `backend/app/service/plugin.go` |
| Service manager | `GET` | `{api}/archive/package/{id}/{version}.tar` | Download plugin archive | `backend/app/service/plugin.go` |
| Compiler service (config `api.compiler.api`) | `POST` | `{api}/compile` or `{api}/ai-compile` | Start compile after SCP of tarball | `backend/app/library/compiler/compiler.go` |
| Autopilot / AI backend (config `api.ai_backend`) | `GET` | `{url}/api/v1/documents/download/{blobPath}` | Download `config.js` for AI projects (`X-API-Key` if set) | `backend/app/library/compiler/compiler.go` |
| Socket / realtime server (config `api.platform.socket_url`) | `POST` | `{socket_url}/broadcast` | Notify channel when build cleanup fails | `backend/app/service/application.go` |

**Non-HTTP:** SCP to hosts configured under `api.compiler.scp` and `api.platform.scp` (`backend/app/library/compiler/compiler.go`, tmp compression).

## 6. Database / Models / Tables

**MariaDB** (typical connection name `mariadb` in code): models in `backend/app/model/*` using `DB("mariadb", ...)`.

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| `applications` | App records | Read / Write | `backend/app/model/app.go` |
| `users` | Users (also hydrated from introspect) | Read | `backend/app/model/user.go` |
| `companies` | Company | Read | `backend/app/model/company.go` |
| `roles` | Roles | Read | `backend/app/model/role.go` |
| `licenses` | License | Read | `backend/app/model/license.go` |
| `applications_type` | App type | Read | `backend/app/model/applications_type.go` |
| `packages` | App-linked JS/CSS packages | Read / Write | `backend/app/model/package.go` |
| `audits` | Audit log | Read / Write | `backend/app/model/audit.go` |
| `Build`, `BuildType`, `Platform`, `Publishing`, `PublishingChannel` | Build/publish metadata | Read / Write | `backend/app/model/build.go`, `build_type.go`, `platform.go`, `publishing.go`, `publishing_channel.go` — **INFERRED** physical table names follow GORM defaults (e.g. `builds`, `build_types`) unless overridden |

**SQLite** (per app, file under project storage, e.g. `data.db`):

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| `eas_page` | Pages | Read / Write | `backend/app/model/page.go` |
| `eas_snippet` | Snippets | Read / Write | `backend/app/model/snippet.go` |
| `eas_custom_style` | Custom styles | Read / Write | `backend/app/model/custom_style.go` |
| `eas_table` | Dynamic tables | Read / Write | `backend/app/model/table.go` |
| `eas_field` | Table fields | Read / Write | `backend/app/model/table_field.go` |
| `eas_record` | Row headers | Read / Write | `backend/app/model/table_record.go` |
| `eas_text_value` / `eas_integer_value` / `eas_numeric_value` | Typed cell values | Read / Write | `backend/app/model/table_*.go` |

## 7. Jobs / Queues / Cron / Workers

No jobs, queues, cron tasks, or workers were found.

Build work is **synchronous HTTP-triggered** (compile request + external compiler pipeline), not an in-repo scheduler.

## 8. Configuration & Environment

Runtime configuration is **JSON files** under the application root’s `config/` directory, resolved via `helper.DirectoryAppRoot()` + `/config/<name>.json` (`backend/app/helper/config.go`). For `go run` from `backend/`, that is `backend/config/`. **No `.env` file is used** as the primary mechanism.

### Primary: `backend/config/*.json` (runtime; templates are `*.sample`)

| Key / Name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `application.port` | HTTP listen port | `application.json.sample`, `backend/main.go` |
| `application.projectStoragePath` | Root for per-app files and SQLite | `application.json.sample`, `backend/app/model/model.go` |
| `application.storagePath` | Backend storage area | `application.json.sample` |
| `application.platformPath` | V5 platform path (compiler integration) | `application.json.sample`, compiler services |
| `application.sessionSecret` | Session signing | `application.json.sample`, session middleware |
| `database.mariadb` / `pgsql` / `sqlite` | DB connections | `database.json.sample`, `backend/app/database/` |
| `auth.defaultOrigin`, `auth.allowedOrigins` | CORS | `auth.json.sample`, `backend/app/middleware/cors.go` |
| `auth.APIKey` | Validates `API-Key` header for config-auth routes; also sent as `API-Key-Platform` to compiler | `auth.json.sample`, `backend/app/middleware/middleware.go`, `backend/app/library/compiler/compiler.go` |
| `api.main_backend`, `api.service_manager`, `api.compiler`, `api.platform`, `api.ai_backend` | Outbound URLs, keys, SCP, socket broadcast | `api.json.sample`, middleware, services, compiler |

### Other / Secondary

| Variable / Key | Purpose | Used In |
|---|---|---|
| — | Root `Makefile` **`make config`** copies `application`, `auth`, `database`, `socket` samples only | `Makefile`; **`api.json` must be created separately** (e.g. copy `backend/config/api.json.sample`) |

**Rules:** Never commit populated `backend/config/api.json` or other runtime JSON with real secrets (see `AGENTS.md`).

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| Main Emobiq backend | HTTP API | OAuth token introspection for Bearer auth | `backend/app/middleware/middleware.go`, `api.json.sample` |
| Service manager (EMSM) | HTTP API | Plugin search, package metadata, archives | `backend/app/service/plugin.go`, `api.json.sample` |
| Compiler service | HTTP + SCP | Receives tarball, runs compile | `backend/app/library/compiler/compiler.go`, `api.json.sample` |
| Autopilot / AI backend | HTTP | Fetch `config.js` for AI builds | `backend/app/library/compiler/compiler.go`, `api.json.sample` |
| Socket / broadcast server | HTTP | Push notifications to clients | `backend/app/service/application.go`, `api.json.sample` |
| MariaDB | Database | Shared relational data | `database.json.sample`, models |
| Per-app SQLite files | Database | Editor/app content | `database.json.sample`, `backend/app/model/model.go` |
| SSH/SCP target hosts | Network | Transfer artifacts | `api.json.sample`, compiler `Tmp.Scp` |

## 10. Main Flows

### Flow: Authenticated API request

1. Client sends `Authorization: Bearer <token>` for routes under `middleware.AuthTokenFull()`.
2. `authenticateToken` calls main backend `POST .../oauth/tokens/introspect` (`backend/app/middleware/middleware.go`).
3. User is stored in session; handlers use `auth.UserSessionFunc` per `backend/app/AGENTS.md`.
4. Controller returns JSON envelope via `result()`.

### Flow: Trigger mobile/web build

1. Client calls `POST /v1/applications/:appid/build` (Bearer).
2. Service prepares project tarball via `backend/app/library/compiler`, may SCP to compiler host, then `POST` to `{compiler.api}/compile` or `/ai-compile`.
3. External compiler/builder may call back `POST /v1/applications/:appid/build/complete` or related endpoints with API-Key (`backend/app/route/setup.go`).
4. Failures may write audits and optionally `POST` `{socket_url}/broadcast` (`backend/app/service/application.go`).

### Flow: Plugin search / editor data

1. `GET /v1/plugins/search` or `/process/editor/data` (Bearer).
2. Service calls service manager (`/search`, `/package`, `/archive/package/...`) (`backend/app/service/plugin.go`).

## 11. Things Other Repos Depend On

- **Stable `/v1` REST routes** and JSON envelope used by clients (editors, tooling) — see §4.
- **Bearer token contract**: introspection against main backend (`/oauth/tokens/introspect`) must remain compatible for session population shape expected in `middleware.authenticateToken`.
- **Build callbacks**: external compiler/builder calls `POST /v1/applications/:appid/build/complete` and `POST /v1/applications/build/clear` with `API-Key` matching `auth.json` (and related build-status behavior).
- **Outbound config contract**: `api.json` keys (`main_backend`, `compiler`, `platform`, `service_manager`, `ai_backend`) are integration points for sibling services.
- **Dual DB layout**: MariaDB metadata + per-app SQLite under `projectStoragePath` / `app/<appid>` — other services must not assume a single database.

## 12. Unknowns / Needs Confirmation

- **WebSocket endpoints:** Handlers exist (`backend/app/controller/socket.go`) but are not mounted in `route/setup.go`; confirm whether any deployment registers them.
- **Exact MariaDB table names** for models without explicit `TableName()` (e.g. `Build`) — labeled **INFERRED**; verify against live schema or migrations if present outside this scan.
