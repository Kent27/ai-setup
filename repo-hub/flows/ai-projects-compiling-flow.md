# AI Projects Compiling Flow

## Purpose

Describe how **AI-created eMOBIQ projects** (produced via the Autopilot stack and surfaced in **emobiq-main-frontend**) are compiled into platform-ready **mobile** artifacts: **emobiq-main-backend** packages sources and invokes **emobiq-compiler**—using the **AI Cordova** pipeline when applicable—then **emobiq-builder** runs native toolchains, with progress and completion reported back through **emobiq-main-backend** and **emobiq-socket**.

**Out of scope for this doc:** NoCode editor flows (**emobiq-client-frontend** / **emobiq-client-backend**) until those repos are profiled in **repo-hub**.

## Known Context

- Compiler exposes **`POST /v1/ai-compile`** for **AI Cordova** only (`compiler_type: cordova`); standard **`POST /v1/compile`** still covers Cordova / React Native / Java Spring Boot for non–AI-compile paths.
- Whether **main-backend** chooses **`/v1/ai-compile`** vs **`/v1/compile`** for a given AI application is **not** spelled out in **repo-hub**’s repo profiles—called out under Unknowns.
- **repo-hub** sources for this narrative: `repo-profiles/*`, `flows/*`.

## Repositories Involved

### `autopilot-ai`

- Responsibilities: AI backend that **generates and iterates** projects (runs, documents, assets). Upstream of “having an AI project” in the platform; not the compile executor itself.
- Key files: Not documented in **repo-hub**—see upstream `docs/repo-profile.md` and `README.md`.

### `emobiq-main-frontend`

- Responsibilities: Authenticated web app including **`/ai/*`** routes and application lifecycle; triggers **build** through **emobiq-main-backend**; WebSocket subscription for build progress; **`REACT_APP_STORAGE_URL`** for generated/static assets where URLs are composed.
- Key files: `frontend/src/common/helper/HttpRequest.ts`, `ImplementControl.ts`, `frontend/src/common/hooks/Socket.ts`, AI and application slices under `frontend/src/common/zustand/slice/`.

### `emobiq-main-backend`

- Responsibilities: **`POST /v1/applications/:application_code/build`** packages sources and calls **`api.compiler`** (`golang/app/library/compiler/compiler.go`); when **`api.compiler.scp`** is configured, **SCP** to the compiler host is part of that handoff; callbacks **`POST /v1/applications/:application_code/build/complete`**, **`POST /v1/logs`**, audits; **`/v1/application-control/*`** for implementation limits (may gate compile-related actions); **`/v1/autopilot/...`** for AI app settings/files/plugin assets **orthogonal to** the compile HTTP contract but part of AI project data.
- Key files: `golang/route/api.go`, `golang/app/library/compiler/compiler.go`, `golang/app/controller/application.go`, `application_control.go`, `autopilot.go`, `audit.go`.

### `emobiq-compiler`

- Responsibilities: **`POST /v1/ai-compile`** — AI Cordova pipeline then same **SCP + `POST /v1/build/application`** handoff to builder; **`POST /v1/compile`** — standard Cordova/React Native/Java Spring Boot path when used for AI apps that do not use `ai-compile`. Uses **`backend/config/api.json`** for main-backend license paths, Connector, Autopilot AI URLs in RN/AI-related code paths. Callbacks to **`emobiq_url`**, **`POST`** **`/broadcast`** on **`socket_url`**, **service manager** **`GET /package?id=...`** for plugins.
- Key files: `backend/main.go`, `backend/app/controller/compiler.go`, `backend/app/library/compiler/*.go` (`ai_compiler.go`, etc.), `backend/app/model/compiler.go`, `backend/config/*.json`.

### `emobiq-builder`

- Responsibilities: **`POST /v1/build/application`** — async native build; platform logs/completion and **`POST`** socket **`/broadcast`**; **SCP** artifact upload. Same contract whether upstream job came from **`/v1/compile`** or **`/v1/ai-compile`**.
- Key files: `backend/app/controller/builder.go`, `backend/app/library/builder/*.go`, `backend/config/*.json`.

### `emobiq-socket`

- Responsibilities: HTTP **`POST …/broadcast`** for compiler/builder producers; WebSocket **`/v1/ws/:channelName`** (registry)—**main-frontend** may use **`{SOCKET_URL}/ws/{channel}`** depending on gateway.
- Key files: See `repo-profiles/emobiq-socket.md` where maintained; otherwise infer from upstream `README.md` and code.

### `emobiq-storage`

- Responsibilities: Static hosting for **generated project assets** tied to the **Autopilot / AI generation** flow. Distinct from compiler/builder SCP staging; used when UIs or APIs reference **`REACT_APP_STORAGE_URL`** paths (e.g. `/app/*`).
- Key files: `index.js` (per `repo-profiles/emobiq-storage.md`).

### `emobiq-service-manager`

- Responsibilities: Cordova/React Native **plugin/package metadata** during compile (**`GET`** package endpoint with `id` query).
- Key files: Upstream repo—not indexed here; base URL from compiler **`application.serviceManagerAPI`** / **`CompilerInput`**.

## End-to-End Flow

1. **Project existence (upstream)**  
   User creates or iterates an **AI project** via **emobiq-main-frontend** ↔ **autopilot-ai** (and related UIs such as **autopilot-debug-agent** iframe for debugging—not the compile orchestrator). Generated artifacts may be reachable via **emobiq-storage** URLs depending on product wiring.

2. **Trigger compile/build (UI → platform)**  
   User starts a **build** from the main web app → **`POST /v1/applications/:application_code/build`** on **emobiq-main-backend** (OAuth). Optional **`POST /v1/application-control/*`** checks may run around implementation limits.

3. **Platform → compiler**  
   **emobiq-main-backend** submits the packaged project to **`api.compiler`**. **`api.compiler.scp`** drives **SCP** of the tarball to the compiler host: if it is present in config, that transfer is required alongside the compile **`POST`**, not a skippable extra.

4. **Compiler accepts job**  
   Validates **`API-Key`** and **`API-Key-Platform`**; returns immediately; work runs in a **goroutine**. Duplicate jobs blocked by **compile locks** on disk.

5. **Compile phase**  
   **AI Cordova:** runs **`ai_compiler.go`** pipeline (only **`compiler_type: cordova`** supported on **`/v1/ai-compile`**). **Standard compile:** Cordova / React Native / Java Spring Boot per **`/v1/compile`**. Fetches plugins via **service manager** where applicable.

6. **Handoff to builder**  
   Tar artifact, **SCP** to builder staging, **`POST {builderAPI}/v1/build/application`** with **`API-Key`** and **`API-Key-Platform`**.

7. **Build phase (builder)**  
   Async native toolchain run; logs and **`/broadcast`** updates; **`POST`** completion to **`emobiq_url`** (`/applications/{id}/build/complete` or legacy v1). Stale locks cleared via **`/applications/build/clear`** where implemented.

8. **Client observation**  
   **emobiq-main-frontend** opens **`WebSocket`** to **`REACT_APP_SOCKET_URL`** and refreshes app state via **main-backend**. Binary or log download links may point at **storage** or other hosts—deployment-specific.

## Endpoints

| From | To | Method | Endpoint | Purpose | Auth |
|---|---|---|---|---|---|
| Browser | `emobiq-main-backend` | `POST` | `/v1/applications/:application_code/build` | Start build → compiler invocation | OAuth Bearer |
| Browser | `emobiq-main-backend` | `POST` | `/v1/application-control/*` | Implementation limits (may include compile gates) | Per route |
| Browser | `autopilot-ai` | Various | `api/v1/...` (representative) | AI project generation/editing (**not** compile trigger) | Per Autopilot contract |
| `emobiq-main-backend` | `emobiq-compiler` | `POST` | **`/v1/ai-compile`** | **AI Cordova** compile job | Configured compiler credentials + platform headers |
| `emobiq-main-backend` | `emobiq-compiler` | `POST` | **`/v1/compile`** | Standard compile (incl. RN / non–AI-compile Cordova) | Same pattern |
| `emobiq-compiler` | Platform (`emobiq_url`) | `POST` | `/api/?controller=logger`, `/logs`, `/applications/{id}/build/complete`, `/applications/build/clear`, legacy v1 variants | Logs, completion, clear | `API-Key-Platform` pattern |
| `emobiq-compiler` | `emobiq-socket` | `POST` | `/broadcast` | Realtime compile status | Compiler `auth.SocketAPIKey` |
| `emobiq-compiler` | `emobiq-builder` | `POST` | `/v1/build/application` | Trigger native build | `API-Key` + `API-Key-Platform` |
| `emobiq-compiler` | `emobiq-service-manager` | `GET` | `/package?id=...` | Plugin/package resolution | Per deployment |
| `emobiq-builder` | Platform (`emobiq_url`) | `POST` | `/logs`, `/applications/{appID}/build/complete`, `/applications/build/clear`, legacy v1 | Logs + completion + stale clear | Outbound platform key |
| `emobiq-builder` | `emobiq-socket` | `POST` | `/broadcast` | Build progress | Builder `auth.keySocketAPI` |
| Browser | `emobiq-socket` | WebSocket | `/ws/:channel` or `/v1/ws/:channelName` | Subscribe (path depends on gateway) | **Needs confirmation** |
| Browser | `emobiq-storage` | `GET` | `/*` (static) | AI/generated asset URLs when used | None in sample code (**needs confirmation** prod) |

## Data / State

| Repo | Table / Model / Redis / Storage | Purpose |
|---|---|---|
| `autopilot-ai` | Upstream persistence (Postgres/blob/etc.) | Runs, documents, generated assets—**not** documented here |
| `emobiq-main-backend` | MariaDB: `applications`, builds-related entities, audits | AI app records and build lifecycle |
| `emobiq-main-backend` | SQLite per application (`data.db`) | Project content packaged into compile tarball |
| `emobiq-main-backend` | Redis (optional) | Not compile-core |
| `emobiq-compiler` | `backend/storage/tmp/`, `lock/`, `files/` | Workspace, locks, templates |
| `emobiq-builder` | `backend/storage/scp/…`, locks | Staged tarballs, builder locks |
| `emobiq-storage` | Filesystem under service root | Served static assets for AI/generated content |

## Config

| Repo | Config / Env Var | Purpose |
|---|---|---|
| `emobiq-main-backend` | `golang/config/api.json` → `compiler` (HTTP + `scp` when defined), `platform`, `ai_backend`, etc. | Compiler handoff, callbacks, socket URL |
| `emobiq-main-backend` | `golang/config/auth.json` | Service **`API-Key`** for callbacks |
| `emobiq-compiler` | `backend/config/application.json` | Builder URL, SCP, `serviceManagerAPI` |
| `emobiq-compiler` | `backend/config/auth.json` | `APIKey`, `BuilderAPIKey`, `SocketAPIKey` |
| `emobiq-compiler` | `backend/config/api.json` | `mainBackendAPI`, `autopilotAIRepoURL`, `autopilotAIAPIKey`, connector URLs—used in AI/RN compiler paths |
| `emobiq-builder` | `backend/config/application.json`, `auth.json` | Toolchains, signing, inbound/outbound keys |
| Per-job JSON | `CompilerInput` fields (`emobiq_url`, `socket_url`, `scp_*`, …) | Route callbacks and staging |
| `emobiq-main-frontend` | `REACT_APP_MAIN_BACKEND_URL`, `REACT_APP_AI_BACKEND_URL`, `REACT_APP_SOCKET_URL`, `REACT_APP_STORAGE_URL`, `REACT_APP_DEBUG_AGENT_URL` | AI + build + realtime + assets + debug iframe |

## Failure Cases

| Case | Expected Behavior | Where Handled |
|---|---|---|
| Duplicate compile (lock held) | Background job may no-op; prevents overlapping compiles | `emobiq-compiler` locks |
| Duplicate builder job | Reject/skip concurrent build | `emobiq-builder` locks |
| Wrong compiler endpoint for AI Cordova | Job may fail or take wrong pipeline | **main-backend** route to **`/v1/ai-compile`** vs **`/v1/compile`** (**needs confirmation**) |
| **`/v1/ai-compile`** with non-Cordova type | Unsupported—handler rejects AI compile path | `ai_compiler.go` |
| SCP / network failure | Logs, socket errors, completion failure | Compiler/builder request helpers |
| Missing **`API-Key`** / **`API-Key-Platform`** | Auth failure | Compiler/builder middleware |

## Debugging Notes

- **`200` on compile/build HTTP** only means the job was **accepted**—track **`/broadcast`**, **`/logs`**, and **`/applications/.../build/complete`**.
- For **AI Cordova**, reproduce with **`POST /v1/ai-compile`** and validate **`compiler_type`** is **`cordova`**.
- Confirm **`api.compiler`** path suffix in **main-backend** matches the intended pipeline (**`ai-compile`** vs **`compile`**).
- Inspect compiler **`storage/lock/`** and builder lock dirs for stuck jobs.

## Unknowns / Needs Confirmation

- **Exact branch in `emobiq-main-backend`** that selects **`/v1/ai-compile`** (AI Cordova) versus **`/v1/compile`** for AI-labelled applications.
- Whether **AI React Native** projects always use **`/v1/compile`** only (likely—**`ai-compile`** is Cordova-only per compiler context).
- **`broadcast`** payload schema and WebSocket **channel** naming for AI project builds.
- How **autopilot-ai** outputs land in **main-backend** packaging paths versus **emobiq-storage** URLs for the same project.
- Production hardening for **emobiq-storage** (auth/TLS) when used for AI artifacts.
