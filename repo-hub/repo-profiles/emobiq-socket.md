# Repo Profile: emobiq-socket

Last updated: 2026-05-07  
Confidence: High

## 1. Purpose

Go service that exposes **authenticated WebSocket channels** and an HTTP **broadcast** API for pushing messages to subscribers. It validates callers via **OAuth token introspection** against a configurable main backend or via **API keys** in local JSON config. Connection state lives **in memory** (hub/channels/clients); MongoDB support exists in the codebase but is **not initialized** from `main.go`.

## 2. Tech Stack

- **Language:** Go (`backend/go.mod`; module directive `go 1.14`)
- **Framework:** Gin, Gorilla WebSocket
- **Runtime:** Linux/macOS native binary; Docker Compose for local stack; backend image built from `backend/Dockerfile` (multi-stage, final `scratch`)
- **Package manager:** Go modules (`backend/go.mod`, `backend/go.sum`)
- **Database:** MongoDB (official Go driver), wired through `database.Initialize()` when used
- **Other important dependencies:** `golang.org/x/crypto`, internal `package/mongo-migrate` (migration CLI), `pkg/errors`

## 3. Important Folders / Files

| Path | Purpose |
|------|---------|
| `backend/main.go` | Gin router: `/v1` WebSocket and broadcast routes |
| `backend/app/controller/socket.go` | WebSocket connect + JSON broadcast handler |
| `backend/app/library/socket/` | Hub, channels, client upgrade (`client.go` origin check, protocol echo) |
| `backend/app/middleware/` | `KeyOrTokenAuth`, OAuth introspection, API-key validation |
| `backend/app/helper/` | JSON config load (`config.go`), `ApiRequest` HTTP client (`request.go`), paths (`directory.go`, `environment.go`) |
| `backend/database/` | MongoDB adapter behind `database.DB` interface |
| `backend/migrations/` | Migration modules registered via `init()` (template in `template_mongodb.go`) |
| `backend/cmd/migrations/main.go` | CLI: `new`, `up`, `down` against MongoDB |
| `backend/config/*.json.sample` | Templates for runtime JSON config (concrete `*.json` gitignored) |
| `docker-compose.yml` / `docker-compose.override.yml` | Scratch backend image + optional local MongoDB |
| `Makefile` | `run`, `stop`, `deploy`, `config` |
| `service/` | Sample systemd unit + deploy shell for compiled `backend/main` |

## 4. Exposed Endpoints

| Method | Path | Purpose | Main handler/file | Auth |
|--------|------|---------|-------------------|------|
| `GET` | `/v1/ws/:channelName` | Upgrade to WebSocket; subscribe to named channel | `backend/main.go` → `controller.SocketConnect` | Yes — `middleware.KeyOrTokenAuth()` |
| `POST` | `/v1/broadcast` | Push message to one channel (or all if `channel` empty) | `backend/main.go` → `controller.SocketBroadcast` | Yes — same middleware |

**Auth mechanism (from code):**

- If `Sec-Websocket-Protocol` is non-empty: treated as **token** path → introspection (see §5); on failure, token checked against `auth.json` **APIKeys** (legacy fallback).
- Else if `API-Key` header present: validated against `auth.json` **APIKeys**.

Successful and error JSON bodies use `{ "success", "data", "message" }` (`backend/app/controller/controller.go`).

## 5. Outbound API Calls

| Target | Method | Endpoint / URL | Purpose | Source file |
|--------|--------|----------------|---------|---------------|
| Main backend (config) | `POST` | `{api.json main_backend.api}/oauth/tokens/introspect` | Validate access token | `backend/app/middleware/oauth.go` via `backend/app/helper/request.go` (`ApiRequest`) |

Base URL is read from `api.json` → `main_backend.api` (`helper.ConfigGet("api", "main_backend")`). With a typical sample base such as `…/v1`, the resolved path is **`POST …/v1/oauth/tokens/introspect`**.

### Main backend: OAuth introspection contract

Canonical behavior on the **main backend** (not defined in this repo):

| Aspect | Detail |
|--------|--------|
| Request | `POST /v1/oauth/tokens/introspect`, header `Authorization: Bearer <token>`, **empty body** |
| Success | HTTP **200**, body `{"success":true,"data":{"user":{...UserResponse...}},"message":"Success"}` |
| Auth failure | HTTP **401**, body `{"success":false,"data":null,"message":...}` |
| Canonical types | `OAuthTokensIntrospectResponse` — main backend `golang/app/resource/oauth.go`; `UserResponse` — main backend `golang/app/resource/user.go` |

**This repo’s caller:** `authenticateToken` always sends a **JSON body** (`token`, `token_type_hint: "access_token"`) and `Content-Type: application/json`, plus the same Bearer header (`backend/app/middleware/oauth.go`). It expects a **200** response and reads `result["data"].(map)["user"].(map)` — consistent with the success shape above.

No other programmatic HTTP clients were found in `backend/` besides `ApiRequest` usage above.

## 6. Database / Models / Tables

| Table / collection | Purpose | Read/Write | Source file |
|--------------------|---------|------------|-------------|
| MongoDB collections (generic) | CRUD via `database.DB` when initialized | Read/Write | `backend/database/mongodb.go` |
| `users` (default name) | **Only** if `BasicAuth()` middleware used: credential lookup | Read | `backend/app/middleware/auth.go` (collection/table name overridable via legacy fields in `auth.json`) |
| `migrations` | Tracks applied migrations | Read/Write | `backend/cmd/migrations/main.go` (`migrate.SetMigrationsCollection`) |

**Note:** `backend/main.go` does **not** call `database.Initialize()`, so the WebSocket server path does not use MongoDB unless changed elsewhere. Migration CLI does initialize DB.

No active ORM models directory; commented sample in `backend/migrations/template_mongodb.go` referenced a `users` collection.

## 7. Jobs / Queues / Cron / Workers

No jobs, queues, cron schedules, or worker processes were found in application runtime code.

| Name | Type | Purpose | Source file |
|------|------|---------|-------------|
| Migration CLI | Manual CLI | `go run cmd/migrations/main.go new|up|down` | `backend/cmd/migrations/main.go` |

## 8. Configuration & environment

### Primary: `backend/config/*.json` (JSON files next to app root)

The app loads integration settings via `helper.ConfigRead()` / `helper.ConfigGet()` from files under `config/` relative to `helper.DirectoryAppRoot()` (`backend/app/helper/config.go`, `backend/app/helper/directory.go`). Allowed names: **`api`**, **`application`**, **`database`**, **`auth`**, **`socket`**. Concrete files are **gitignored**; templates are **`backend/config/*.json.sample`**.

| Key / name | Purpose (no secret values) | Evidence |
|------------|----------------------------|----------|
| `application.port` | HTTP listen port (default `8080` if missing) | `backend/main.go`, `backend/config/application.json.sample` |
| `application.dateFormat` | App-level date format string | `backend/config/application.json.sample` |
| `main_backend.api` | Base URL for token introspection (`/oauth/tokens/introspect` appended) | `backend/config/api.json.sample`, `backend/app/middleware/oauth.go` |
| `APIKeys` | Map of API key → consumer description for `API-Key` header auth | `backend/config/auth.json.sample`, `backend/app/middleware/auth_config.go` |
| `socket.checkOrigin`, `socket.allowed` | WebSocket origin gate: hostnames without port | `backend/config/socket.json.sample`, `backend/app/library/socket/client.go` |
| `database.*` | Mongo connection when DB used (`database`, `host`, `port`, `name`, `username`, `password`) | `backend/config/database.json.sample`, `backend/database/database.go` |

**Secondary mechanisms:**

- **`helper.EnvAppCompiled`** (`backend/app/helper/environment.go`): defaults **`false`** in source (typical for `go run` / cwd-based `./config`). **Production sets this to `true`** (confirmed): app root is the executable’s directory, so `config/` resolves next to the deployed binary (aligned with compiled/container layouts).
- **Docker Compose**: mounts `./backend/config` → `/config` and `./backend/log` → `/log` for service `orangekloud-scratch` (`docker-compose.yml`).

No `.env`-first loading was found for these settings.

## 9. Service Dependencies

| Dependency | Type | Why needed | Evidence |
|------------|------|------------|----------|
| Main backend API | HTTP (`POST` introspection) | Validate bearer-style tokens | `backend/app/middleware/oauth.go` |
| MongoDB | Database | Optional: migrations; `BasicAuth` user lookup if that middleware were used | `backend/database/`, `backend/cmd/migrations/main.go` |
| Calling browsers/clients | WebSocket + HTTP | Consume `/v1/ws/...` and (for servers) POST `/v1/broadcast` | `backend/main.go` |

## 10. Main Flows

### Flow: Authenticated WebSocket subscribe

1. Client requests `GET /v1/ws/:channelName` with either `Sec-Websocket-Protocol: <token>` or `API-Key: <key>`.
2. `KeyOrTokenAuth` validates token (introspection + optional API-key fallback) or API key map.
3. `SocketConnect` resolves hub/channel, starts channel goroutine if new, upgrades connection (`client.go`), echoes `Sec-Websocket-Protocol` on upgrade.
4. Client messages are broadcast within the channel (`readPump` → `channel.broadcast`).

### Flow: Server-side broadcast

1. Authenticated `POST /v1/broadcast` with JSON `{ "channel", "message", "created_at?" }` (`SocketBroadcastInput`).
2. If `created_at` set, message may be wrapped or merged into JSON (`backend/app/controller/socket.go`).
3. `hub.Broadcast` sends to named channel or all channels if `channel` is empty.

### Flow: Token validation (introspection)

1. Read `api.json` base URL; `POST` `{base}/oauth/tokens/introspect` (see §5 for canonical main-backend contract vs this client’s JSON body).
2. On **200**, parse `data.user` and set Gin auth user context (`backend/app/middleware/oauth.go`).
3. On introspection failure, retry by treating the token string as an `APIKeys` entry (**legacy**, noted in code).

## 11. Things Other Repos May Depend On

- **Stable HTTP routes:** `GET /v1/ws/:channelName`, `POST /v1/broadcast`.
- **WebSocket contract:** Auth via `Sec-Websocket-Protocol` (token) or separate `API-Key` header; same protocol header echoed on upgrade (clients relying on subprotocol negotiation).
- **CORS/origin behavior:** Allowed hosts list in `socket.json` (`allowed`) matched to request host without port.
- **Introspection API:** Main backend `POST /v1/oauth/tokens/introspect` and response envelopes as documented in §5; shared shapes `OAuthTokensIntrospectResponse` / `UserResponse` in main backend `golang/app/resource/oauth.go` and `golang/app/resource/user.go`.
- **Broadcast JSON shape:** `{ "success", "data", "message" }` on HTTP responses.
