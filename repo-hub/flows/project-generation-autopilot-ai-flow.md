# Project Generation in Autopilot AI

## Purpose

Describe how an **AI application project** is created and driven through the **autopilot-ai** generation pipeline: authenticated clients (typically **emobiq-main-frontend**) call the **autopilot-ai** FastAPI surface to create **projects** and **runs**, the service validates access via **emobiq-main-backend**, may enforce **credits** via **orangekloud-subscription** (reachable through env named like SSO in the Autopilot service), spawns a **MetaGPT / Hydra** subprocess for multi-agent generation, and persists state in **PostgreSQL** and **Azure Blob Storage**. This doc stops at **generated Autopilot artifacts and platform-side app records** where profiles document them—**native compile** is covered in **`repo-hub/flows/ai-projects-compiling-flow.md`**.

## Known Context

- **User-provided:** Flow is about **project generation** in the **autopilot-ai** pipeline and which repos participate; repos of interest include **autopilot-ai**, **main-frontend**, **main-backend**, **subscription**, and **sso**.
- **Boundary:** **`orangekloud-sso`** does **not** appear in **autopilot-ai**’s documented outbound HTTP list; it is part of the **authentication prerequisite** (tokens later introspected by **emobiq-main-backend**—see **`repo-hub/flows/login-flow.md`**).
- **`SSO_BASE_URL` in autopilot-ai:** Env name in **autopilot-ai** refers to the base used for **credits** HTTP (`/api/credits/*`); **`repo-profiles/autopilot-ai.md`** describes this as SSO / subscriptions API—**orangekloud-subscription** exposes those paths (**NEEDS CONFIRMATION** that every deployment points `SSO_BASE_URL` at subscription, not another host).

## Related Docs

- Service map: `repo-hub/maps/service-map.md`
- Database map: `repo-hub/maps/database-map.md`
- Login / tokens (prerequisite): `repo-hub/flows/login-flow.md`
- Downstream compile: `repo-hub/flows/ai-projects-compiling-flow.md`

## Repositories Involved

### `autopilot-ai`

- Repo profile: `repo-hub/repo-profiles/autopilot-ai.md`
- Responsibilities:
  - Exposes **`/api/v1`** REST for **projects**, **runs**, **documents**, **progress**, **pricing**, **chats**, **integration**, **checkpoints**, **templates**, etc. (see **`docs/fast_api.md`** in that repo).
  - Authenticates requests with **`X-API-Key`** or **`Authorization: Bearer`**, validating Bearer tokens via **`POST /v1/oauth/tokens/introspect`** on **emobiq-main-backend**.
  - **Creates** Autopilot **`project`** (and related) rows; starts **runs** via **`POST /api/v1/runs/`** (prefix from profile); launches **`python -m main`** subprocess with Hydra overrides (**`subprocess_manager.py`**).
  - For **non-`local`** environments, interacts with **`SSO_BASE_URL`** for **`/api/credits/balance`**, **`/api/credits/hold`**, **`/api/credits/charge`**, **`/api/credits/release-hold`** (**`private_pricing_api.py`**); balance may be considered before run validation (**`RunInput`** in **`models/query/runs.py`** per profile).
  - Reads/writes **PostgreSQL** ORM tables (`project`, `run`, `progress`, …); stores workspace/checkpoint assets in **Azure Blob**; may call **main-backend** for external API payloads and SSO user API keys, **service-manager** for plugin archives, **simulation** HTTP API, and **Supabase** management APIs per outbound table.
- Key files:
  - `src/entrypoint_api.py`, `src/orangekloud_copilot_fastapi/routers/v1/projects_routes.py`, `src/orangekloud_copilot_fastapi/routers/v1/runs_routes.py`, `src/orangekloud_copilot_fastapi/subprocess_manager.py`, `src/orangekloud_copilot_fastapi/services/auth.py`
  - `src/main.py`, `src/orangekloud_copilot/modeling/utilities/private_pricing_api.py`, `conf/` (Hydra)

### `emobiq-main-frontend`

- Repo profile: `repo-hub/repo-profiles/emobiq-main-frontend.md`
- Responsibilities:
  - Presents **`/ai/...`** and related UX; drives AI workspace state via **axios** against **`REACT_APP_AI_BACKEND_URL`** ( **`api/v1/...`** patterns per profile ).
  - Resolves optional **per-project VM / load balancer** URL via **`GET api/v1/projects/{id}/server`** (**`getLoadBalancerUrl`** / **`AiSlice.ts`**).
  - Relies on **OAuth session** (Bearer in cookies) from the normal platform login flow when calling backends.
  - **`REACT_APP_DEBUG_AGENT_URL`:** embeds **autopilot-debug-agent** iframe for debugger UX—not the sole path for generation, but can call Autopilot (**`POST`** **`/api/autopilot/import`** etc. per debug-agent profile).
- Key files: `frontend/src/common/helper/HttpRequest.ts`, `frontend/src/common/zustand/slice/AiSlice.ts`, `frontend/src/Route.tsx`, AI pages under `frontend/src/page/`

### `emobiq-main-backend`

- Repo profile: `repo-hub/repo-profiles/emobiq-main-backend.md`
- Responsibilities:
  - **`POST /v1/oauth/tokens/introspect`:** Autopilot-ai (and other clients) validate end-user Bearer tokens.
  - **Registry & keys:** **`GET /v1/external_apis/{ext_api_id}/content`**, **`GET /v1/sso/user/{idp_user_id}/api-keys`** used from Autopilot for connector/external API preprocessing (**cross-repo profile evidence**—not exhaustive of all `/v1` surfaces).
  - **Platform AI app surface:** **`/v1/autopilot/...`** (settings/files/plugin assets per profile) and **applications** routes that include **AI project create** (logical group in **`route/api.go`** table—“exact handler ordering vs Autopilot **`/api/v1/projects`** **NEEDS CONFIRMATION**”).
  - **`api.ai_backend`** in **`golang/config/api.json`** for outbound calls to the AI/autopilot backend from Go services (**`golang/app/service/autopilot.go`**, **`golang/app/library/ai/backend.go`**).
- Key files: `golang/route/api.go`, `golang/app/controller/autopilot.go`, `golang/app/controller/application.go`, `golang/app/middleware/oauth.go`, `golang/config/api.json`

### `orangekloud-subscription`

- Repo profile: `repo-hub/repo-profiles/orangekloud-subscription.md`
- Responsibilities:
  - Hosts **`GET /api/credits/balance`**, **`POST /api/credits/hold`**, **`POST /api/credits/charge`**, **`POST /api/credits/release-hold`** (**auth:** Bearer and/or **`API-Key`** + IP allowlist patterns per credits routes in profile)—these match the **`SSO_BASE_URL`** client usage described in **autopilot-ai**.
  - **orthogonal** Domain data: **`credits`**, **`credit_hold_amounts`**, subscription attributes—not the Autopilot Postgres schema.
- Key files: `routes/api.php`, `app/Http/Controllers/Api/CreditsController.php`, middleware for token/API key/IP

### `orangekloud-sso`

- Repo profile: `repo-hub/repo-profiles/orangekloud-sso.md`
- Responsibilities:
  - **Prerequisite only** for flows that use **OAuth2 Bearer** tokens Minted/session-established via Passport + platform OAuth (**login-flow**): Autopilot-ai never calls SSO HTTP directly per **autopilot-ai** profile scans.
  - IdP **`users`** and **`oauth_*`** tables underpin tokens that ultimately pass **main-backend introspection**.
- Key files: (see SSO profile—**not** on the runtime Autopilot request path beyond token issuance upstream)

### `autopilot-debug-agent` (related UI; optional for minimal create+run)

- Repo profile: `repo-hub/repo-profiles/autopilot-debug-agent.md`
- Responsibilities: Embedded debugger calls Autopilot HTTP via **`{AUTOPILOT_AI_URL}`** (**includes `/api/v1`**) for **`projects/`**, **`runs/`**, **`documents/`**, **`pricing/`**, etc.; **`POST /api/autopilot/import`** for project import.
- Key files: `app/lib/external/autopilot.ts`, `app/routes/api.autopilot.import.ts`

## End-to-End Flow

1. **Authenticate (prerequisite)**  
   End user obtains a session acceptable to Go APIs: typically **OAuth** via **`emobiq-main-frontend`** → **`emobiq-main-backend`** → **`orangekloud-sso`** (**`login-flow`**). The browser then attaches **`Authorization: Bearer`** from cookies when calling backends (**`HttpRequest.ts`**).

2. **Open AI workspace in main-frontend**  
   User navigates to **`/ai/...`** (guarded routes per **main-frontend** profile). SPA loads AI state via **`REACT_APP_AI_BACKEND_URL`** (`api/v1/...`).

3. **Create Autopilot project (HTTP)**  
   Client calls **`autopilot-ai`** project routes (**`projects_routes.py`**—profile: “Various **`/api/v1/projects/...`**” including template-based create). Service writes **`project`** / **`project_details`** (and related) in **PostgreSQL** and interacts with **Azure Blob** where applicable (**architecture / utilities**).

4. **Start a run**  
   Client calls **`POST /api/v1/runs/`** (**`runs_routes.py`**). Copilot persists **`run`** and related rows and starts **`python -m main`** with Hydra configuration (**`subprocess_manager.py`**).

5. **AuthZ on Autopilot requests**  
   **`auth.py`** calls **`POST {MAIN_BACKEND_BASE_URL}/v1/oauth/tokens/introspect`** for Bearer tokens; **`X-API-Key`** path uses **`FASTAPI_API_KEY`** per profile.

6. **Credits (non-local)**  
   When **`ENVIRONMENT`** is not **`local`**, **`PrivatePricingAPI`** may **`GET`** **`{SSO_BASE_URL}/api/credits/balance`** and use hold/charge/release during pricing/ops (**exact call sites:** **`private_pricing_api.py`** and **`RunInput`** validation per profile—not every step enumerated here).

7. **Agent pipeline**  
   **`src/main.py`** / MetaGPT agents read/write workspace, blob storage, **`progress`**, **`chat_history`**, **`cost`**, **`documents`**, etc., until completion or failure (**INFERRED** partial failure handling from architecture notes).

8. **Platform-side AI application (overlap—NEEDS CONFIRMATION)**  
   **emobiq-main-backend** exposes **`/v1/autopilot/...`** and **applications** routes including **AI project create** (**profile table**). The strict ordering (“Autopilot **`project`** first vs MariaDB **`applications`** row first”) is **NOT** spelled out in **repo-hub**—treat as **NEEDS CONFIRMATION**.

## Endpoints

| From | To | Method | Endpoint | Purpose | Auth / Headers | Confidence |
|---|---|---|---|---|---|---|
| Browser | `emobiq-main-frontend` | **Various** | `REACT_APP_AI_BACKEND_URL` + `api/v1/...` (e.g. `projects/…`, documents, runs patterns) | AI workspace UX | Bearer (cookies) **INFERRED** from `HttpRequest` patterns | High |
| Browser | `emobiq-main-frontend` | `GET` | **`api/v1/projects/{id}/server`** (**INFERRED** path fragment) | Resolve load-balancer VM URL for project | Bearer | Medium |
| `autopilot-ai` | `emobiq-main-backend` | `POST` | **`/v1/oauth/tokens/introspect`** | Validate Bearer token | Service config + token | High |
| `autopilot-ai` | `emobiq-main-backend` | `GET` | **`/v1/external_apis/{ext_api_id}/content`** | External API registry JSON | **`MAIN_BACKEND_API_KEY`** (+ request context per preprocessor) | High |
| `autopilot-ai` | `emobiq-main-backend` | `GET` | **`/v1/sso/user/{idp_id}/api-keys`** | Connector/API key lookup | **`MAIN_BACKEND_API_KEY`** | High |
| Browser or service | `autopilot-ai` | **Various** | **`/api/v1/projects/...`** | Project CRUD, downloads, template create, modes | `X-API-Key` and/or Bearer | High |
| Browser or service | `autopilot-ai` | **`POST`** | **`/api/v1/runs/`** | Start run → subprocess (`python -m main`) | `X-API-Key` and/or Bearer | High |
| Browser or service | `autopilot-ai` | **Various** | **`/api/v1/pricing/...`**, **`/api/v1/progress/...`**, **`/api/v1/documents/...`**, etc. | Billing, progress, artifacts | Authenticated (**`authenticate`**) unless public image exception | High |
| `autopilot-ai` | **`orangekloud-subscription`** (via `SSO_BASE_URL`) | `GET` | **`/api/credits/balance`** | Credit balance | Bearer / keys per subscription credits routes (**NEEDS CONFIRMATION** mapping env→host) | High |
| `autopilot-ai` | **`orangekloud-subscription`** (via `SSO_BASE_URL`) | `POST` | **`/api/credits/hold`**, **`/api/credits/charge`**, **`/api/credits/release-hold`** | Holds and charges | Per **CreditsController** auth | High |
| `autopilot-ai` | **Simulation backend** (**INFERRED** separate deploy) | **Mixed** | `{SIMULATION_BASE_URL}/v1/project/create`, `/v1/project/{id}`, `/v1/project/{id}/api/{db}/{table}` (**per profile**) | eMOBIQ backend simulation CRUD during generation | **`SIMULATION_API_KEY`** | Medium |
| `autopilot-ai` | `emobiq-service-manager` | `GET` | **`/api/archive/package/{slug}/{version}.tar`** | Plugin package archive | **`SERVICE_MANAGER_API_KEY`** | High |
| **`autopilot-debug-agent`** | `autopilot-ai` | Mixed | Paths under **`{AUTOPILOT_AI_URL}`** ending in **`projects/`**, **`runs/`**, **`documents/`**, **`pricing/`**, … | Debugger-driven Autopilot ops | Bearer / Autopilot HTTP contract | High |
| `emobiq-main-backend` | **AI backend** (`api.ai_backend`) | **GET/POST** | Deployment-defined (**`backend.go`** / **`autopilot.go`**) | Platform-side autopilot orchestration (**not fully enumerated** in hub) | OAuth / internal | Medium |

## Data / State

| Repo | Table / Model / Redis / Queue / Storage / File | Purpose | Read / Write | Confidence |
|---|---|---|---|---|
| `autopilot-ai` | PostgreSQL **`project`**, **`project_details`**, **`run`**, **`progress`**, **`document`**, **`cost`**, **`cost_details`**, **`chat_history`**, **`human_input`**, **`supabase_credentials`**, **`progress_notes`**, **`plugins`** | Core generation state | Read / Write | High |
| `autopilot-ai` | PostgreSQL **`templates`**, **`template_bookmarks`**, **`plugin_credentials`** | Templates / bookmarks / plugin secrets | Read / Write | High |
| `autopilot-ai` | **Azure Blob** (`AZURE_STORAGE_CONNECTION_STRING`) | Workspace, checkpoints, templates | Read / Write | High |
| `autopilot-ai` | **Subprocess** + **Hydra **`conf/`** | Agent run configuration | Read | High |
| `emobiq-main-backend` | MariaDB **`applications`** (+ related platform tables); **SQLite** per-app **`data.db`** | Packaged IDE app record + content (**AI project create**, later compile tarball) | Read / Write | High |
| `emobiq-main-backend` | Optional **Redis** | Connector rate limiting | **Not** Autopilot Postgres path | High |
| `orangekloud-subscription` | **`credits`**, **`credit_hold_amounts`** | Credit balance / holds surfaced to Autopilot | Read / Write | High |
| `orangekloud-sso` | **`users`**, **`oauth_*`** (Passport) | Token issuance prerequisite | Read / Write (IdP lifecycle) | High |
| `emobiq-main-frontend` | Browser memory / cookies | Session + SPA state only | Read / Write | High |

## Config

| Repo | Config / Env Var | Purpose | Confidence |
|---|---|---|---|
| `autopilot-ai` | **`MAIN_BACKEND_BASE_URL`**, **`MAIN_BACKEND_API_KEY`** | Introspection + platform API pulls | High |
| `autopilot-ai` | **`FASTAPI_API_KEY`** | **`X-API-Key`** + Swagger basic (**profile**) | High |
| `autopilot-ai` | **`SSO_BASE_URL`**, **`SSO_API_KEY`**, **`SSO_OAUTH_URL`** | Credits / SSO-named integration (**typically subscription** credits) | High |
| `autopilot-ai` | **`ENVIRONMENT`** | **`local`** vs billed behavior (**profile**) | High |
| `autopilot-ai` | **`METAGPT_*`** | LLM provider endpoints, keys, model | High |
| `autopilot-ai` | **`DB_*`**, **`AZURE_STORAGE_CONNECTION_STRING`** | PostgreSQL + blob | High |
| `autopilot-ai` | **`SERVICE_MANAGER_URL`**, **`SERVICE_MANAGER_API_KEY`** | Plugin tarballs | High |
| `autopilot-ai` | **`SIMULATION_BASE_URL`**, **`SIMULATION_API_KEY`** | Backend simulation REST | Medium |
| `autopilot-ai` | **`CONNECTOR_V1_URL`**, **`CONNECTOR_V2_URL`**, **`VM_PUBLIC_IP`**, **`VM_HOST_IP`**, **`VM_HOST_PORT`** | Embedded connector JS / ops metadata (**`VM_HOST_*` usage NEEDS CONFIRMATION** per profile) | Medium–Low |
| `emobiq-main-frontend` | **`REACT_APP_AI_BACKEND_URL`**, **`REACT_APP_MAIN_BACKEND_URL`**, **`REACT_APP_DEBUG_AGENT_URL`** | AI API + auth + iframe | High |
| `emobiq-main-backend` | **`golang/config/api.json`** → **`ai_backend`**, **`sso`**, etc. | Outbound AI backend + OAuth | High |
| `orangekloud-subscription` | **`API_KEY`**, **`WHITELISTED_IPS`**, Bearer validation for credits | Matches Autopilot’s credit client assumptions | Medium |

## Failure Cases

| Case | Expected Behavior | Where Handled | Confidence |
|---|---|---|---|---|
| Invalid / expired Bearer on Autopilot route | **`401`** / auth rejection | **`autopilot-ai`** **`auth.py`** + introspection | High |
| **emobiq-main-backend** unreachable during introspect | Request fails (**NEEDS CONFIRMATION** status body) | Autopilot **`auth`** | Medium |
| Insufficient credits (non-local) | Run/pricing rejection or billing errors (**exact UX NEEDS CONFIRMATION**) | **`private_pricing_api.py`**, validators, **`/api/v1/pricing/...`** | Medium |
| Subprocess crash | Run failure / zombie reconciliation | **`ProcessManager`**, **`prune_zombies`** lifespan (**`entrypoint_api.py`**) | High |
| **`ENVIRONMENT=local`** | Credits behavior skipped/different (**profile**) | **`private_pricing_api.py`** | High |
| DB / blob outage | Persist or workspace failure | DB + **`azure_blob_storage.py`** | Medium |

## Debugging Notes

- Prove auth path: **`POST /v1/oauth/tokens/introspect`** from Autopilot container to **main-backend** with same token the browser sends.
- Confirm **`SSO_BASE_URL`** in Autopilot resolves to the deployment that exposes **`/api/credits/*`** (**subscription** profile).
- Watch **`POST /api/v1/runs/`** then **process manager** logs for **`python -m main`**; inspect **`progress`**, **`run`**, **`cost`** tables.
- **`prune_zombies`** at startup (**`lifespan`**) reconciles orphaned processes vs DB (**`entrypoint_api.py`** / **`ops_routes.py`** per profile).
- Daily **MetaGPT log cleanup** (**`schedule`**, **`log_remover.py`**) unrelated to correctness but noisy if misconfigured.

## Unknowns / Needs Confirmation

- Order and coupling between **Autopilot `project` creation** and **main-backend `applications` / AI project create** routes.
- Full **method/path** inventory for **`GET /api/v1/projects`** sub-routes (**`docs/fast_api.md`** upstream is authoritative).
- Whether **`SSO_BASE_URL`** always denotes **orangekloud-subscription** vs a separate SSO host for credits only.
- Whether **compiled production** installs use additional queues (**Redis/Celery**) outside **`src/`** scan (**autopilot-ai** profile gap).
- How **generated outputs** sync into **SQLite `data.db` / tarball** consumed by **`ai-projects-compiling-flow.md`** (**partially overlapping** Unknown in that flow doc).

---

## Suggested follow-up updates (maps)

- **`repo-hub/maps/service-map.md`:** Refresh the **`autopilot-ai`** row (it currently says autopilot-ai is **not profiled**); add an **Important Cross-Repo Flows** entry linking **`repo-hub/flows/project-generation-autopilot-ai-flow.md`** (generation) distinct from **`ai-projects-compiling-flow.md`** (compile). Add **`emobiq-main-frontend` → `autopilot-ai`** relationship with evidence **`repo-profiles/emobiq-main-frontend.md`** + **`repo-profiles/autopilot-ai.md`**. Optionally note **`orangekloud-sso`** as indirect (tokens only).

- **`repo-hub/maps/database-map.md`:** The **Autopilot PostgreSQL** section already lists **`project`** / **`run`** / **`progress`**. Add a cross-reference sentence that **`project-generation-autopilot-ai-flow.md`** owns the narrative from **HTTP create → subprocess → Postgres/blob** (no DDL change unless new tables emerge).
