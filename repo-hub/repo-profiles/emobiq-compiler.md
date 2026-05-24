# Repo Profile: emobiq-compiler
Last updated: 2026-05-06  
Confidence: High

## 1. Purpose
Backend service (**eMOBIQ Compiler Server**) that accepts compile jobs over HTTP, expands **Cordova**, **React Native**, or **Java Spring Boot** apps from uploaded project tarballs and templates, then **SCP-transfers** artifacts to a remote **builder** and triggers the builder via HTTP. An **AI** variant pipeline exists for **AI Cordova** (`/v1/ai-compile`). Embedded assets live under `backend/storage/files/**` for generated projects.

## 2. Tech Stack
- Language: Go (module declares `go 1.14`; README mentions newer local Go for dev)
- Framework: [Gin](https://github.com/gin-gonic/gin)
- Runtime: Standalone HTTP process (`backend/main.go`); optional Docker (`docker-compose.yml`, `backend/Dockerfile`)
- Package manager: Go modules (`backend/go.mod`)
- Database: MongoDB client ([mongo-driver](https://pkg.go.dev/go.mongodb.org/mongo-driver/mongo)) — **only used by migration CLI** in normal operation (`cmd/migrations`); user/BasicAuth paths are dead code unless someone wires them later (see §6, §12)
- Other important dependencies: Gorilla WebSocket (socket client/helper code), SCP/SSH tooling via helpers, YAML/plist libs for mobile config

## 3. Important Folders / Files
| Path | Purpose |
|---|---|
| `backend/main.go` | Gin router; only `/v1` compile routes; startup lock cleanup |
| `backend/app/controller/` | HTTP handlers (`compiler.go`, `controller.go`; `socket.go`, `users.go` present but **not** registered in `main.go`) |
| `backend/app/middleware/` | `ConfigAuth()` (API key from config); `BasicAuth()` (Mongo-backed; unused by current router) |
| `backend/app/library/compiler/` | Orchestration (`compiler.go`), platform compilers, AI variants, locks |
| `backend/app/helper/` | Config load (`config.go`), HTTP client (`api.go`), logging, SCP, archives, downloads |
| `backend/database/` | Mongo adapter + `Initialize()` (used by migrations CLI and legacy unused auth/user snippets) |
| `backend/config/*.json.sample` | Committed templates for runtime JSON configs (`make config` copies most) |
| `backend/cmd/migrations/main.go` | Mongo migrate CLI (`up`/`down`) |
| `backend/storage/` | `files/` templates, `tmp/` workspace, `lock/` compile locks, SCP staging |
| `docker-compose.yml` | Backend image, volume mounts for `config`, `log`, `storage`, `keys/.ssh` |
| `Makefile` | `run`, `stop`, `config` (copies sample configs) |

## 4. Exposed Endpoints
Endpoints provided by this repo (from `backend/main.go`). All live under **`/v1`** and require header **`API-Key`** validated against `backend/config/auth.json` when `APIKey` is set (`middleware.ConfigAuth` allows empty/disabled config key).

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `POST` | `/v1/compile` | Start standard compile (Cordova / React Native / Java Spring Boot) | `backend/app/controller/compiler.go` (`Compile`) | Yes (`API-Key` config) + **`API-Key-Platform`** (required by handler) |
| `POST` | `/v1/ai-compile` | Start AI Cordova compile pipeline | `backend/app/controller/compiler.go` (`AICompile`) | Same as above |

**Not wired in router (dead code paths):** `controller.UserCreate`, `controller.SocketConnect` / `SocketBroadcast` — not used by the running API; leftover from the framework scaffold / README samples and **could be deleted in a future cleanup** (not removed yet).

## 5. Outbound API Calls
| Target Service / Host | Method | Endpoint / URL pattern | Purpose | Source file |
|---|---|---|---|---|
| eMOBIQ platform (caller-supplied base) | `POST` | `{emobiq_url}/api/?controller=logger` (v1) | Audit log during compile | `backend/app/library/compiler/compiler.go` |
| eMOBIQ platform | `POST` | `{emobiq_url}/logs` (v2) | Logs with tags | `compiler.go` |
| eMOBIQ platform | `POST` | `{emobiq_url}/api/?controller=build&action=complete` (v1); `{emobiq_url}/applications/{app_id}/build/complete` (v2) | Mark build failed / completion path | `compiler.go` |
| eMOBIQ platform | `POST` | `{emobiq_url}/api/?controller=build&action=clear` (v1); `{emobiq_url}/applications/build/clear` (v2 default) | Clear stuck build session | `compiler.go` (`platformBuildClear`) |
| Socket service (caller-supplied base) | `POST` | `{socket_url}/broadcast` | Push compile status to frontend channel (`API-Key` from `auth.SocketAPIKey`) | `compiler.go` |
| Builder service | `POST` | `{builderAPI from application.json}/build/application` | Trigger remote build after SCP | `compiler.go` (`build`) |
| Service Manager | `GET` | `{serviceManagerAPI from application.json}/package?id=...` | Resolve Cordova/React Native plugin packages | `cordova.go`, `react_native.go`, `plugin.go` |
| URLs from `api.json` | `GET` / `POST` | e.g. connector/autopilot/backend paths assembled in code | AI Cordova and RN flows (repo URLs, license path names in sample) | `ai_cordova.go`, `react_native.go` |
| Arbitrary URLs | `GET` | Full URL passed in | Asset/plugin downloads | `backend/app/helper/file.go` (`DownloadFile`), `ai_cordova.go` |

Base URLs for platform/socket/SCP come largely from **`CompilerInput`** JSON (`backend/app/model/compiler.go`), not only from server config.

## 6. Database / Models / Tables
| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| MongoDB collections (generic) | Helper CRUD API used by **`cmd/migrations`**; same package would back `middleware.BasicAuth` / `UserCreate` if wired (they are **not**) | Migration CLI (+ unused snippets) | `backend/database/mongodb.go` |
| `users` (collection name constant) | Only referenced from **unregistered** `UserCreate` / `BasicAuth` | Not used by live routes | `backend/app/model/user.go` |

Mongo **migration metadata** lives in whatever collection the vendored [`mongo-migrate`](https://github.com/xakep666/mongo-migrate) package uses (see `backend/package/mongo-migrate/`).

**Note:** `database.Initialize()` is called from **`backend/cmd/migrations/main.go` only.** `backend/main.go` does **not** open Mongo—the HTTP server does **not** use the DB. **Confirmed:** deployments do **not** add DB init or user/socket routes here; unused code remains for now.

## 7. Jobs / Queues / Cron / Workers
| Name | Type | Purpose | Source file |
|---|---|---|---|
| Compile job | In-process goroutine | `Compile` / `AICompile` spawn background work; HTTP returns immediately | `backend/app/library/compiler/compiler.go`, `ai_compiler.go` |
| Lock cleanup | Startup | Remove stale compile lock files | `backend/main.go` (`compiler.CleanLockFiles`) |

No separate queue workers, cron schedules, or job daemons were found in-repo.

## 8. Configuration & environment

### Primary: `backend/config/*.json` (JSON on disk; `helper.ConfigRead` / `ConfigGet`)

The server reads fixed filenames under the app root (`helper.DirectoryAppRoot()` → `"."` when not compiled): `backend/config/application.json`, `database.json`, `auth.json`, `socket.json`, and **`api.json`** where AI/autopilot-related compiler code needs it (`ConfigRead("api")`). **`make config`** copies `.sample` files for application, auth, database, socket; **`api.json` is not copied** — copy from `backend/config/api.json.sample` or maintain a local `backend/config/api.json` (prefer **gitignore** / secrets handling for environments with real keys).

| Key / name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `application.port` | HTTP listen port | `backend/config/application.json.sample`; `backend/main.go` |
| `application.builderAPI`, `scpUser`, `scpHost`, `scpPort`, `scpDir` | Builder URL + SCP defaults for uploads | `application.json.sample`; `helper`/compiler SCP usage |
| `application.serviceManagerAPI` | Plugin/package resolution base URL | `application.json.sample`; `cordova.go`, `react_native.go`, `plugin.go` |
| `application.deleteTmpFile` | Temp artifact cleanup toggle | `application.json.sample`; `compiler.go` (`deleteTmpFile`) |
| `auth.APIKey` | Validates incoming **`API-Key`** header when non-empty | `auth.json.sample`; `middleware/auth_config.go` |
| `auth.BuilderAPIKey` | Sent to builder as **`API-Key`** | `auth.json.sample`; `compiler.go` (`build`) |
| `auth.SocketAPIKey` | Sent to socket **`/broadcast`** | `auth.json.sample`; `compiler.go` |
| `socket.checkOrigin`, `socket.allowed` | WebSocket origin policy (socket **library**/external server coordination) | `socket.json.sample` |
| `database.*` | Mongo connection (`database`, `host`, `port`, `name`, `username`, `password`) | `database.json.sample`; `database/database.go` |
| `api.mainBackendAPI`, `mainBackendAPIKey`, `mainBackendLicenseAPIPath`, `connectorRepoURL`, `connectorCallbackURL`, `autopilotAIRepoURL`, `autopilotAIAPIKey`, `serviceManagerAPIKey`, etc. | Base URLs + auth headers/paths for main backend license checks, Connector, Autopilot AI, Service Manager keyed calls where those features run | Actual values live in local `backend/config/api.json`; key inventory in `api.json.sample`; consumers `ai_cordova.go`, `react_native.go` |

### Other (secondary)
| Variable / key | Purpose | Used in |
|---|---|---|
| `EnvAppCompiled` | Switches config path resolution vs `.` cwd | `backend/app/helper/environment.go`, `directory.go` |
| Docker bind mounts | Injects host `backend/config`, `storage`, `log`, SSH keys | `docker-compose.yml` |

**Rules respected:** No secret values pasted above.

## 9. Service Dependencies
| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| eMOBIQ platform API | HTTP | Logging, build status, clear-session calls | `compiler.go` |
| Builder API | HTTP + SCP drop | Consume tarball and drive native builds | `compiler.go`; `sendToBuilder` |
| Socket notification service | HTTP (`/broadcast`) | Real-time UI updates | `compiler.go` |
| Service Manager HTTP API | HTTP | Plugin/package metadata | `cordova.go`, `react_native.go` |
| Remote SCP host | SSH/SCP | Upload compiled tarball (`API-Key-Platform` flows) | Compiler input + `helper.ScpUploadFile` |
| MongoDB | Database | **Migration CLI only** for this codebase’s HTTP server | `database/`; `cmd/migrations/main.go` |
| External git/npm/cordova assets | Network / tooling | Cordova RN templates, plugin sources (during compile) | `storage/files/**`, `plugin.json` |

## 10. Main Flows

### Flow: Standard compile (`POST /v1/compile`)
1. Handler validates **`API-Key-Platform`** and JSON **`CompilerInput`** (`compiler.go` controller).
2. `compiler.Compile` rejects if lock exists for app/build; starts **goroutine**.
3. Extract `{domain}.{app_id}.tar.gz` under `backend/storage/tmp/`, load `emobiq.json`.
4. Run platform compiler (Cordova / React Native / Java Spring Boot) using templates under `backend/storage/files/`.
5. Tar artifact, **SCP** to builder path driven by input + defaults.
6. **POST** `{builderAPI}/build/application` with builder + platform headers; emit logs/socket messages on failure paths.

### Flow: AI compile (`POST /v1/ai-compile`)
1. Same HTTP/auth pattern as standard compile (`AICompile`).
2. Only **`compiler_type`: `cordova`** supported in `AICompile` (`ai_compiler.go`); runs **AI Cordova** pipeline then same builder handoff.

### Flow: Mongo migration (CLI, not HTTP)
1. `go run cmd/migrations/main.go up|down` calls `database.Initialize()` and applies vendored migration runner.

## 11. Things Other Repos May Depend On
- **Stable HTTP contract:** `POST /v1/compile` and `POST /v1/ai-compile` with JSON body matching `model.CompilerInput` and headers **`API-Key`** (server) + **`API-Key-Platform`** (platform identity for downstream calls).
- **Callback behavior:** Compiler calls caller-supplied **`emobiq_url`**, **`socket_url`**, and SCP targets — platform and builder repos must expose those endpoints and accept SCP layout as implemented.
- **`/build/application` request shape** Builder service must accept the payload assembled in `compiler.go` (`build` function).
- **Artifact layout:** tarball naming and extraction paths (`compiler.go`, `extract` / tmp dir helpers).

## 12. Unknowns / Residual Notes
- **Mongo + user/socket code:** **`database.Initialize()` is not called from `main.go`**, and **`UserCreate` / WebSocket Gin handlers are unregistered.** Team confirmation: deployments **do not** rely on those paths. Treat as unused scaffold; **eligible for removal later** once someone deletes the helpers and README samples (**no deletion done in repo yet**).
- **Mongo prod shape:** Outside migration metadata, irrelevant for this service while the HTTP process does not call `database.Initialize()` (confirmed above).
