# Repo Profile: emobiq-builder
Last updated: 2026-05-06
Confidence: High

## 1. Purpose
This repository hosts a Go-based build orchestration service that receives app build requests, runs Cordova/React Native/Java Spring Boot build pipelines, uploads build artifacts, and reports status back to upstream platform services.

## 2. Tech Stack
- Language: Go
- Framework: Gin (HTTP API), Gorilla WebSocket (library present)
- Runtime: Go binary on Linux/macOS, plus Dockerized Ubuntu build environment
- Package manager: Go modules (`backend/go.mod`)
- Database: MongoDB driver integrated (main runtime path appears build-focused; DB is used by migration tooling and optional auth flows)
- Other important dependencies: SSH/SCP command-line tools, Cordova, Node/NPM/NPX, Gradle, Xcode tooling, Maven wrapper

## 3. Important Folders / Files
| Path | Purpose |
|---|---|
| `backend/main.go` | API entrypoint; registers routes and starts server |
| `backend/app/controller/builder.go` | Validates build request and dispatches async builder job |
| `backend/app/library/builder/` | Core build orchestration and compiler-specific build logic |
| `backend/app/library/builder/request.go` | Outbound calls to platform APIs and socket broadcast API |
| `backend/app/library/builder/lock.go` | Build lock files + startup cleanup/recovery callbacks |
| `backend/app/helper/config.go` | Canonical runtime config loader for `config/*.json` |
| `backend/config/*.json.sample` | Canonical committed config templates for runtime/integrations |
| `backend/database/` | MongoDB abstraction used by migration and optional auth/user flows |
| `backend/cmd/migrations/main.go` | Migration CLI for MongoDB migrations collection |
| `docker-compose.yml` | Local runtime wiring and config/log/storage mounts |
| `Makefile` | Bootstrap commands to create config files and run/stop via Docker |
| `eMOBIQ Builder Server.postman_collection.json` | API usage examples (supporting artifact) |

## 4. Exposed Endpoints
Endpoints provided by this repo.

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `POST` | `/v1/build/application` | Accept build request and start async build processing | `backend/main.go`, `backend/app/controller/builder.go` | Yes (`API-Key` via `ConfigAuth`; also requires `API-Key-Platform`) |

Dormant in this repo (`backend/main.go` does not wire them):
- **`users` / basic-auth sample routes**: Handlers and middleware live under `backend/app/controller/users.go` and `backend/app/middleware/auth.go`; canonical HTTP surface for auth/users is **another repo**.
- **`/ws/:channelName` / in-process `/broadcast` WebSocket handlers**: Code exists (`backend/app/controller/socket.go`, `backend/app/library/socket/*`) but routes are **not registered here**. The realtime **socket server** is **another repo**; **this repo only POSTs outbound** builds status to `<socket_url>/broadcast` (`backend/app/library/builder/request.go`).

## 5. Outbound API Calls
External/internal endpoints this repo calls.

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| `input.EmobiqURL` | `POST` | `/api/?controller=logger` (v1) | Write build log/audit events to platform | `backend/app/library/builder/request.go` |
| `input.EmobiqURL` | `POST` | `/logs` (v2) | Write build log/audit events to platform | `backend/app/library/builder/request.go` |
| `input.EmobiqURL` | `POST` | `/api/?controller=build&action=complete` (v1) | Mark build completion status | `backend/app/library/builder/request.go` |
| `input.EmobiqURL` | `POST` | `/applications/{appID}/build/complete` (v2) | Mark build completion status | `backend/app/library/builder/request.go` |
| `input.EmobiqURL` | `POST` | `/applications/build/clear` | Clear stale in-progress build status at startup recovery | `backend/app/library/builder/request.go`, `backend/app/library/builder/lock.go` |
| `input.SocketURL` | `POST` | `/broadcast` | Broadcast build progress/status to app channel | `backend/app/library/builder/request.go` |
| `input.ScpHost` over SSH/SCP | `SSH`/`SCP` | `mkdir -p <scp_directory>`, upload artifact to `<user>@<host>:<directory>/<file>` | Create remote target dir and transfer packaged build output | `backend/app/helper/file.go`, `backend/app/helper/scp.go` |

## 6. Database / Models / Tables
| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| `users` (`model.TableUser`) | User account model for basic auth/registration helper flows | Read/Write | `backend/app/model/user.go`, `backend/app/controller/users.go`, `backend/app/middleware/auth.go` |
| `migrations` collection | Tracks applied DB migrations | Read/Write | `backend/cmd/migrations/main.go` |
| MongoDB generic collections | Generic CRUD abstraction used by repository helpers | Read/Write | `backend/database/mongodb.go` |

Note: **`main.go` does not initialize MongoDB or register user/basic-auth routes** for this deployed service shape. **`users`/`BasicAuth`-related CRUD stays in codebase for reuse elsewhere; canonical user API is another repo**.

## 7. Jobs / Queues / Cron / Workers
| Name | Type | Purpose | Source file |
|---|---|---|---|
| Builder execution goroutine | Worker (in-process async task) | Runs compiler-specific build pipeline without blocking HTTP request | `backend/app/controller/builder.go` |
| Lock cleanup fan-out | Worker (startup parallel tasks) | On startup, process stale lock folders and clear platform build state | `backend/app/library/builder/lock.go` |
| Migration command | Manual job/CLI | Apply or rollback MongoDB migrations | `backend/cmd/migrations/main.go` |

No scheduler/cron framework or queue broker was found.

## 8. Configuration & environment
Integration/runtime settings are primarily file-based JSON configs loaded at runtime via `helper.ConfigRead(...)`, not `.env` files.

### Primary: `backend/config/*.json` (+ committed `*.sample`)
This is primary because app code explicitly reads `config/application.json`, `config/auth.json`, `config/database.json`, and `config/socket.json` via `backend/app/helper/config.go`.

| Key / name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `application.port` | HTTP listen port | `backend/config/application.json.sample`, `backend/main.go` |
| `application.iosKeychain`, `iosKeychainPassword`, `iosProvisioningProfileFolder` | iOS signing/keychain/provisioning behavior during builds | `backend/config/application.json.sample`, `backend/app/library/builder/cordova.go`, `backend/app/library/builder/react_native.go` |
| `application.android*Result*`, `ios*Result*`, `*VersionCordova` | Override output paths and platform version behavior for build artifacts | `backend/config/application.json.sample`, `backend/app/library/builder/cordova.go`, `backend/app/library/builder/react_native.go` |
| `application.clientReactNativeNodePath`, `clientJavaHomePath`, `serverJavaHomePath`, `builderHomePath` | Toolchain env overrides used by React Native / Java Spring Boot build commands | `backend/config/application.json.sample`, `backend/app/library/builder/react_native.go`, `backend/app/library/builder/java_spring_boot.go` |
| `auth.keyAPI` | API key required by inbound `ConfigAuth` middleware | `backend/config/auth.json.sample`, `backend/app/middleware/auth_config.go` |
| `auth.keySocketAPI` | API key attached to outbound socket broadcast API calls | `backend/config/auth.json.sample`, `backend/app/library/builder/request.go` |
| `database.database/host/port/name/username/password` | MongoDB connection config | `backend/config/database.json.sample`, `backend/database/database.go`, `backend/database/mongodb.go` |
| `socket.checkOrigin`, `socket.allowed` | WebSocket origin policy for **`app/library/socket` client/upgrader** (in-process WS not wired to router in `main.go`; separate socket repo owns the WS server). | `backend/config/socket.json.sample`, `backend/app/library/socket/client.go` |

### Other (secondary)
| Variable / key | Purpose | Used in |
|---|---|---|
| Build request body fields (`emobiq_url`, `socket_url`, `scp_host`, `scp_username`, etc.) | Per-request integration endpoints and transfer targets | `backend/app/library/builder/builder.go`, `backend/app/controller/builder.go` |
| Header `API-Key` | Inbound API auth to this service | `backend/app/middleware/auth_config.go` |
| Header `API-Key-Platform` | Outbound auth credential when calling platform API | `backend/app/controller/builder.go`, `backend/app/library/builder/request.go` |
| Docker volume mounts (`./backend/config`, `./backend/storage`, `./key/.ssh`) | Supplies runtime config, build artifacts, and SSH keys into container | `docker-compose.yml` |

## 9. Service Dependencies
Services, repos, databases, queues, storage, or external systems this repo appears to depend on.

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| **emobiq-compiler** (external repo/service) | Caller / orchestrator | **Canonical production caller** of `POST /v1/build/application` | Team confirmation |
| eMOBIQ platform backend (`emobiq_url`) | API | Receive build logs, completion callbacks, and clear-build signals | `backend/app/library/builder/request.go` |
| Socket service (`socket_url`, separate repo) | API | Outbound **`POST …/broadcast`** for build progress and final status (WS server not in this repo) | `backend/app/library/builder/request.go`, Team confirmation |
| Remote artifact host (`scp_host`) | Storage / Transfer endpoint | Receive tarred build outputs through SCP | `backend/app/helper/scp.go`, `backend/app/helper/file.go` |
| MongoDB | Database | Supports CRUD abstraction, optional auth/user flows, and migration command | `backend/database/mongodb.go`, `backend/cmd/migrations/main.go` |
| Local/host build toolchains (Cordova, Android SDK, Node, Xcode, Maven) | Build infrastructure | Execute platform-specific compilation/signing commands | `backend/app/library/builder/*.go`, `backend/Dockerfile` |
| SSH key material (`key/.ssh`) | Auth infrastructure | Authenticate SSH/SCP to remote servers | `docker-compose.yml` |

## 10. Main Flows
### Flow: Build request to artifact delivery
1. **emobiq-compiler** (production) invokes `POST /v1/build/application` with `API-Key` and `API-Key-Platform`.
2. Controller validates required payload and normalizes defaults (`compiler_type`, `build_type`, `platform`).
3. Service prevents duplicate in-progress builds via lock-file check.
4. Selected builder (`cordova`, `react-native`, or `java-spring-boot`) runs asynchronously.
5. Builder extracts source tarball from `backend/storage/scp/<compiler>/...`, builds artifacts, then archives results.
6. Artifacts are uploaded to remote storage over SCP.
7. Build status/logs are pushed to platform API and socket broadcast endpoint.

### Flow: Startup stale-lock recovery
1. Service startup calls lock cleanup routine.
2. Cleanup scans lock directories and reads per-domain lock metadata.
3. Service calls platform clear endpoint (`/applications/build/clear`) for stale sessions.
4. Stale lock files/folders are removed.

### Flow: DB migration (manual operator flow)
1. Operator runs `go run cmd/migrations/main.go up|down|new`.
2. Command initializes DB connection and migration collection.
3. Migrations are applied/rolled back and tracked in MongoDB.

## 11. Things Other Repos May Depend On
- Stable build trigger contract: `POST /v1/build/application` with required JSON fields and headers (**primary caller in production**: **emobiq-compiler**).
- Outbound callback contract expected by upstream platform APIs (`/logs`, `/applications/{id}/build/complete`, `/applications/build/clear` and legacy v1 endpoints).
- Socket broadcast payload pattern for build updates (`channel`, `message`, `created_at`) sent to `<socket_url>/broadcast`.
- Artifact packaging naming convention (tar files based on domain/app/platform/build type) used for SCP handoff (INFERRED from builder code).

## 12. Unknowns / Needs Confirmation
- NEEDS CONFIRMATION: **`backend/cmd/migrations/main.go`** (plus `backend/migrations/*`, **`backend/package/mongo-migrate/`**, and `database` config for that CLI) **might be unused** in current ops: nothing in **`main.go`**, **Makefile**, or **CI** runs it; `backend/migrations/` only ships a template with **no registered migrations**. **`database.json` / samples** aligned with “NOT USED - IGNORE” are plausible. **Removing this stack** should be validated with the team before deletion.
