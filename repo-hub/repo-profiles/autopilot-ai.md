# Repo Profile: autopilot-ai

Last updated: 2026-05-07
Confidence: High

## 1. Purpose

Autopilot-ai is eMOBIQ’s **AI application-generation service**: multi-agent NLP workflows (built on MetaGPT) produce specs, UI, code, and related artifacts; a **FastAPI** layer exposes project/run lifecycle, document storage integration, checkpoints, integrations (Supabase, plugins, external APIs), and billing hooks. Generated assets and checkpoints are synced with **Azure Blob Storage**; operational state lives in **PostgreSQL**.

## 2. Tech Stack

- Language: Python (service and agent pipeline); JavaScript snippets embedded/generated for connectors (see `src/orangekloud_copilot/modeling/utilities/api_wrapper.py`).
- Framework: FastAPI (`src/entrypoint_api.py`); Hydra-configured MetaGPT/agent pipeline (`src/main.py`, `conf/`).
- Runtime: Python 3 (exact pin **NEEDS CONFIRMATION** — no root `pyproject.toml`/`requirements.txt` in repo root); Uvicorn for the API (**INFERRED** from docstring in `src/entrypoint_api.py`).
- Package manager: **NEEDS CONFIRMATION** — `metagpt/` subtree ships `requirements.txt`; root app dependencies not centralized in one lockfile observed at repo root.
- Database: PostgreSQL (via SQLAlchemy; connection string from env in `src/orangekloud_copilot/modeling/db_utils/database_manager.py`).
- Other important dependencies: Azure Storage SDK (`src/orangekloud_copilot/modeling/utilities/azure_blob_storage.py`), MetaGPT (vendored under `metagpt/`), `schedule` for in-process daily job (`src/entrypoint_api.py`).

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `src/entrypoint_api.py` | FastAPI app assembly, lifespan, cron setup, `/healthz`, `/docs`, `/redoc`. |
| `src/orangekloud_copilot_fastapi/` | Routers, auth, subprocess/process manager for runs. |
| `src/orangekloud_copilot/` | Agents, actions, DB utilities, Azure/connector/supabase helpers. |
| `src/main.py` | Hydra CLI entry for the long-running MetaGPT/agent pipeline (`python -m main`). |
| `conf/` | Hydra YAML: agents, actions, prompts, `config.yaml`. |
| `metagpt/` | Upstream MetaGPT framework and examples (**INFERRED**: vendored dependency). |
| `docs/fast_api.md` | Route-group documentation ( complements OpenAPI). |
| `docs/system-architecture.md` | Deployment and flow-oriented architecture notes. |
| `.env.example` | Primary integration/runtime env template for this service. |
| `.env.migration.example` | Alternate env naming for migrations / privileged storage (**INFERRED** from filename). |
| `deploy/readme.md` | Ansible-oriented VM deployment notes. |
| `frontend/` | Minimal workspace (`package.json` only in tree scanned). |
| `mock_server_backendgpt/` | Local mock backend (**usage scope NEEDS CONFIRMATION**). |

## 4. Exposed Endpoints

All `/api/v1/*` routes use `Depends(authenticate)` unless noted: **`X-API-Key`** or **`Authorization: Bearer`** (OAuth2 token introspection against main backend — `src/orangekloud_copilot_fastapi/services/auth.py`).

Path prefix for versioned API: **`/api/v1`** (`src/orangekloud_copilot_fastapi/routers/routes.py`). Document routes use the **v2** documents router mounted under the same prefix (v1 documents router is imported but **not** mounted).

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/healthz` | Load balancer / health | `src/entrypoint_api.py` | No |
| `GET` | `/` | Redirect to Swagger | `src/entrypoint_api.py` | No |
| `GET` | `/docs` | Swagger UI | `src/entrypoint_api.py` | Yes (HTTP Basic for Swagger: `authenticate_swagger`) |
| `GET` | `/redoc` | ReDoc | `src/entrypoint_api.py` | Yes (HTTP Basic for Swagger) |
| Various | `/api/v1/projects/...` | Project CRUD, download, template-based create, cost, mode | `src/orangekloud_copilot_fastapi/routers/v1/projects_routes.py` | Yes |
| Various | `/api/v1/runs/...` | Create/list runs, stop, human review, debug agent | `src/orangekloud_copilot_fastapi/routers/v1/runs_routes.py` | Yes |
| Various | `/api/v1/documents/...` | Blob download/upload/release, human review (v2 router) | `src/orangekloud_copilot_fastapi/routers/v2/documents_routes.py` | Yes (see note) |
| Various | `/api/v1/progress/...` | Run/progress read/write, resume options | `src/orangekloud_copilot_fastapi/routers/v1/progress_routes.py` | Yes |
| Various | `/api/v1/pricing/...` | Balance, expected cost, apply charge | `src/orangekloud_copilot_fastapi/routers/v1/pricing_routes.py` | Yes |
| Various | `/api/v1/chats/...` | Chat history, marks, human input, MCP | `src/orangekloud_copilot_fastapi/routers/v1/chats_routes.py` | Yes |
| Various | `/api/v1/integration/...` | Supabase, plugins, external API toggles/credentials | `src/orangekloud_copilot_fastapi/routers/v1/integration_routes.py` | Yes |
| Various | `/api/v1/static/...` | Static template/full-flow assets | `src/orangekloud_copilot_fastapi/routers/v1/static_routes.py` | Yes |
| Various | `/api/v1/checkpoints/...` | Git-style commit/peek/rollback/diff | `src/orangekloud_copilot_fastapi/routers/v1/checkpoints_routes.py` | Yes |
| Various | `/api/v1/ops/...` | Ops (e.g. active runs, VM info) | `src/orangekloud_copilot_fastapi/routers/v1/ops_routes.py` | Yes |
| Various | `/api/v1/codes/...` | Patch and lint helpers | `src/orangekloud_copilot_fastapi/routers/v1/codes_routes.py` | Yes |
| `GET` | `/api/v1/prompts/git-diff` | Prompt template for git diff | `src/orangekloud_copilot_fastapi/routers/v1/prompts_routes.py` | Yes |
| Various | `/api/v1/templates/...` | Template catalog and bookmarks | `src/orangekloud_copilot_fastapi/routers/v1/templates_routes.py` | Yes |
| Various | `/api/v1/devtools/...` | Local-only dev introspection routes | `src/orangekloud_copilot_fastapi/routers/v1/devtools_routes.py` | Yes |

**Note (no auth):** `GET /api/v1/documents/image/download/...` and any path **`startswith`** `/api/v1/documents/image/download` bypass API auth (`src/orangekloud_copilot_fastapi/const.py`, `src/orangekloud_copilot_fastapi/services/auth.py`).

**Detail:** Full method-by-method listing is inlined in route modules above; **`docs/fast_api.md`** summarizes groups.

## 5. Outbound API Calls

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| Main backend (`MAIN_BACKEND_BASE_URL`) | `POST` | `/v1/oauth/tokens/introspect` | Validate Bearer tokens | `src/orangekloud_copilot_fastapi/services/auth.py` |
| Main backend (`MAIN_BACKEND_BASE_URL`) | `GET` | `/v1/external_apis/{ext_api_id}/content` | External API JSON content for masking/prompts | `src/orangekloud_copilot/modeling/utilities/external_api_preprocessor.py` |
| Main backend (`MAIN_BACKEND_BASE_URL`) | `GET` | `/v1/sso/user/{idp_id}/api-keys` | Resolve connector-related API keys | `src/orangekloud_copilot/modeling/utilities/external_api_preprocessor.py` |
| SSO / subscriptions API (`SSO_BASE_URL`) | `GET` | `/api/credits/balance` | Credit balance | `src/orangekloud_copilot/modeling/utilities/private_pricing_api.py` |
| SSO / subscriptions API (`SSO_BASE_URL`) | `POST` | `/api/credits/hold`, `/api/credits/charge`, `/api/credits/release-hold` | Hold, charge, release credits | `src/orangekloud_copilot/modeling/utilities/private_pricing_api.py` |
| Simulation service (`SIMULATION_BASE_URL` + `/v1`) | `POST`/`GET`/`PATCH`/`DELETE` | `/project/create`, `/project/{id}`, `/project/{id}/api/{db}/{table}` | eMOBIQ backend simulation CRUD | `src/orangekloud_copilot/modeling/utilities/emobiq_backend.py` |
| Service Manager (`SERVICE_MANAGER_URL`) | `GET` | `/api/archive/package/{slug}/{version}.tar` | Plugin package archive | `src/orangekloud_copilot/modeling/utilities/plugin_utils.py` |
| Supabase Management API | `GET`/`POST` | `https://api.supabase.com/v1/projects/{project_id}/api-keys?reveal=true` (and related paths) | Keys, OpenAPI, types, query | `src/orangekloud_copilot/modeling/utilities/supabase_utils.py` |
| LLM provider | **INFERRED** | Configured via `METAGPT_*` env | Model calls from MetaGPT pipeline | `src/orangekloud_copilot_fastapi/subprocess_manager.py`, agent config |
| Azure Blob Storage | SDK (not HTTP table) | N/A | Project workspace, templates, checkpoints | `src/orangekloud_copilot/modeling/utilities/azure_blob_storage.py` |

**Browser-side (generated JS, not server HTTP):** Connector service URLs from `CONNECTOR_V1_URL` / `CONNECTOR_V2_URL` are embedded in `REQUEST_JS` in `src/orangekloud_copilot/modeling/utilities/api_wrapper.py` for client-side `fetch` to the connector proxy pattern.

## 6. Database / Models / Tables

ORM models are generic per-table (`src/orangekloud_copilot/modeling/db_utils/table.py`); logical PostgreSQL tables (from `src/orangekloud_copilot/modeling/db_utils/conf.py`):

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| `document` | Document metadata (**note:** architecture doc suggests underused) | Read / Write | `src/orangekloud_copilot/modeling/db_utils/document_db.py` |
| `progress` | Per-run agent/action progress | Read / Write | `src/orangekloud_copilot/modeling/db_utils/progress_db.py` |
| `project` | Project records | Read / Write | `src/orangekloud_copilot/modeling/db_utils/project_db.py` |
| `project_details` | Extended project/server metadata | Read / Write | `src/orangekloud_copilot/modeling/db_utils/project_details_db.py` |
| `run` | Run lifecycle | Read / Write | `src/orangekloud_copilot/modeling/db_utils/run_db.py` |
| `cost` / `cost_details` | Cost tracking | Read / Write | `src/orangekloud_copilot/modeling/db_utils/cost_db.py`, `cost_details_db.py` |
| `chat_history` | Chat transcript | Read / Write | `src/orangekloud_copilot/modeling/db_utils/chat_history_db.py` |
| `human_input` | Human-in-the-loop payloads | Read / Write | `src/orangekloud_copilot/modeling/db_utils/human_input_db.py` |
| `supabase_credentials` | Supabase linkage | Read / Write | `src/orangekloud_copilot/modeling/db_utils/supabase_credentials_db.py` |
| `progress_notes` | Notes linked to progress/chat flows | Read / Write | `src/orangekloud_copilot/modeling/db_utils/progress_notes_db.py` |
| `plugins` / `plugin_credentials` | Plugin selections and secrets | Read / Write | `src/orangekloud_copilot/modeling/db_utils/plugins_db.py`, `plugin_credentials_db.py` |
| `templates` / `template_bookmarks` | Template catalog and user bookmarks | Read / Write | `src/orangekloud_copilot/modeling/db_utils/templates_db.py`, `template_bookmarks_db.py` |

## 7. Jobs / Queues / Cron / Workers

| Name | Type | Purpose | Source file |
|---|---|---|---|
| Log cleanup schedule | In-process cron (`schedule` library, daily midnight) | Delete MetaGPT run logs past retention | `src/entrypoint_api.py`, `src/script_utils/log_remover.py` |
| `prune_zombies` | Startup hook (`lifespan`) | Reconcile DB run state vs live processes | `src/entrypoint_api.py`, `src/orangekloud_copilot_fastapi/routers/v1/ops_routes.py` |
| Run subprocess | Worker-like | Each run: `python -m main ...` subprocess managed by `ProcessManager` | `src/orangekloud_copilot_fastapi/subprocess_manager.py`, `src/orangekloud_copilot_fastapi/routers/v1/runs_routes.py` |

No separate Celery/RQ/redis queue codebase identified in prioritized scan (**NEEDS CONFIRMATION** if used outside `src/`).

## 8. Configuration & Environment

Describe where integration/runtime settings live in this repo.

Use one or both subsections as appropriate.

### Primary: `.env` (loaded via `python-dotenv`)

Dotenv loads `../.env` relative to **`src/`** from `database_manager.py`, `azure_blob_storage.py`, and `src/entrypoint_api.py`. **`.env.example`** is the canonical template for integration-focused keys.

| Key / Name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `ENVIRONMENT` | Toggles behaviours (e.g. skip billing in `local`) | `.env.example`, `src/orangekloud_copilot/modeling/utilities/private_pricing_api.py` |
| `METAGPT_*` | LLM provider base URL, key, model, version, pricing | `.env.example`, `src/orangekloud_copilot_fastapi/subprocess_manager.py` |
| `DB_HOST`, `DB_NAME`, `DB_USERNAME`, `DB_PASSWORD`, `DB_PORT` | PostgreSQL | `.env.example`, `src/orangekloud_copilot/modeling/db_utils/database_manager.py` |
| `AZURE_STORAGE_CONNECTION_STRING` | Blob storage | `.env.example`, `src/orangekloud_copilot/modeling/utilities/azure_blob_storage.py` |
| `SSO_BASE_URL`, `SSO_API_KEY`, `SSO_OAUTH_URL` | Credits/pricing API and SSO-related base | `.env.example`, `src/orangekloud_copilot/modeling/utilities/private_pricing_api.py` |
| `MAIN_BACKEND_BASE_URL`, `MAIN_BACKEND_API_KEY` | OAuth introspection, external API registry | `.env.example`, `src/orangekloud_copilot_fastapi/services/auth.py`, `src/orangekloud_copilot/modeling/utilities/external_api_preprocessor.py` |
| `CONNECTOR_V1_URL`, `CONNECTOR_V2_URL` | Connector base URLs injected into generated JS | `.env.example`, `src/orangekloud_copilot/modeling/utilities/api_wrapper.py` |
| `SIMULATION_BASE_URL`, `SIMULATION_API_KEY` | eMOBIQ server simulation | `.env.example`, `src/orangekloud_copilot/modeling/utilities/emobiq_backend.py` |
| `SUPABASE_CLIENT_ID`, `SUPABASE_CLIENT_SECRET` | Supabase management OAuth client | `.env.example`, `src/orangekloud_copilot/modeling/db_utils/supabase_credentials_db.py` |
| `SERVICE_MANAGER_URL`, `SERVICE_MANAGER_API_KEY` | Plugin archives / service manager | `.env.example`, `src/orangekloud_copilot/modeling/utilities/plugin_utils.py` |
| `FASTAPI_API_KEY` | API key for service + Swagger basic password | `.env.example`, `src/orangekloud_copilot_fastapi/services/auth.py` |
| `VM_PUBLIC_IP` | Exposed host metadata for project details / ops | `.env.example`, `src/orangekloud_copilot_fastapi/routers/v1/ops_routes.py`, `src/orangekloud_copilot_fastapi/services/create_project.py` |
| `VM_HOST_IP`, `VM_HOST_PORT` | Present in `.env.example`; active server usage in current `src/` **NEEDS CONFIRMATION** (see deprecated docs) | `.env.example` |

### Other / Secondary

| Variable / Key | Purpose | Used In |
|---|---|---|
| Hydra `conf/config.yaml` + `conf/agents/*.yaml` | Agent roster, pipeline defaults, multi-LLM file paths | `src/main.py` (**INFERRED** entry pattern) |
| `.env.migration.example` | DB/storage split per environment names | Repo root |

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| PostgreSQL | Database | Project/run/chat/progress state | `src/orangekloud_copilot/modeling/db_utils/database_manager.py` |
| Azure Blob Storage | Storage | Workspace files, checkpoints, templates | `docs/system-architecture.md`, `azure_blob_storage.py` |
| Main backend | Auth + external API registry | Token introspection, external API metadata | `auth.py`, `external_api_preprocessor.py` |
| SSO / subscriptions (`SSO_BASE_URL`) | API | Credits: balance, hold, charge, release | `private_pricing_api.py` |
| Simulation service | API | emobiq backend project/data API during generation | `emobiq_backend.py` |
| Service Manager | API | Plugin packages | `plugin_utils.py` |
| Supabase (management + project) | API | Backend-in-browser mode, schema/types | `supabase_utils.py`, integration routes |
| Connector service | API (client-side) | Proxied external HTTP from generated apps | `api_wrapper.py` |
| LLM providers (Azure OpenAI, etc.) | API | Agent pipeline | `.env.example` `METAGPT_*`, `metagpt/` |
| MetaGPT framework | Library | Multi-agent orchestration | `metagpt/`, `src/main.py` |

## 10. Main Flows

### Flow: Create project and start a run

1. Client creates a project via `/api/v1/projects` (and related template endpoint as needed) — `projects_routes.py`.
2. Client calls `POST /api/v1/runs/` with idea/modes/options — `runs_routes.py`.
3. FastAPI records run in DB and starts **`python -m main`** with Hydra overrides — `subprocess_manager.py`, `runs_routes.py`.
4. Pipeline agents read/write workspace and Azure Blob; persist progress/chat/cost rows — **`src/main.py`**, modeling actions.
5. Run completion triggers cleanup/refund paths per pipeline utilities — (**INFERRED** partial) `terminate_process`/cost flows referenced in architecture doc.

### Flow: Credits for non-local runs

1. Before run validation, balance may be checked via SSO credits API — `RunInput` validator in `src/orangekloud_copilot_fastapi/models/query/runs.py`.
2. `PrivatePricingAPI` holds/charges/releases against `SSO_BASE_URL` — `private_pricing_api.py`, used from pricing endpoints and ops (`release_held_credit`).

### Flow: OAuth for end users on API routes

1. Request includes `Authorization: Bearer <token>`.
2. Copilot calls main backend introspection; on success attaches `request.state.user` — `auth.py`.

## 11. Things Other Repos Depend On

- **`/api/v1` REST contract** — stable prefix and route groups consumed by eMOBIQ clients (see `docs/fast_api.md`).
- **`X-API-Key` + optional Bearer** authentication behaviour and allowlisted public image download path — `auth.py`, `const.py`.
- **`/healthz`** for load balancer probes — `docs/system-architecture.md`, `entrypoint_api.py`.
- **Generated `config.js` / connector patterns** embedding `CONNECTOR_V1_URL`, `CONNECTOR_V2_URL`, and `window.CONFIG` Supabase keys — `api_wrapper.py`, `plugin_utils.py`, `write_code.py` (**INFERRED** consumer: generated apps).
- **`postMessage` contract `OAUTH_CODE_RECEIVED`** (OAuth popup flow in generated `REQUEST_JS`) — `api_wrapper.py`.
- **Plugin archive layout** expected from Service Manager tar structure — `plugin_utils.py`.

## 12. Unknowns / Needs Confirmation

- **NEEDS CONFIRMATION:** Root-level Python dependency lockfile or install path for production (only `metagpt/requirements.txt` observed under subtree).
- **NEEDS CONFIRMATION:** Whether `VM_HOST_IP` / `VM_HOST_PORT` are still used by any active code path (only `.env.example` + deprecated docs).
- **NEEDS CONFIRMATION:** Purpose and deployment of `frontend/` and `mock_server_backendgpt/` in current workflows.
- **NEEDS CONFIRMATION:** Whether an external queue (Redis, etc.) exists outside the scanned `src/` tree.