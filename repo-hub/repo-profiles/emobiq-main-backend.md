# Repo Profile: emobiq-main-backend

Last updated: 2026-05-05

Confidence: Medium (large surface area; exhaustive outbound URL paths vary by deployment and config). Layout: application code and Docker build context use **`golang/`** (not `backend/`).

## 1. Purpose

Central HTTP API (“main backend”) for the eMOBIQ platform: user/company/administration data, OAuth/SSO, applications (CRUD, build, publish, deploy), integrations (compiler, publisher, plugins, AI/autopilot, Supabase proxy, external APIs/cookies), support tickets, auditing, and static asset hosting optionally in development.

## 2. Tech Stack

- **Language:** Go (`golang/go.mod`; module path `bitbucket.org/orangekloud/emobiq-main-backend`)
- **Framework:** Gin (`github.com/gin-gonic/gin`)
- **Runtime:** Compiled binary (`golang/main.go`); listens on `application.port` from config (default `8080`)
- **Package manager:** Go modules (`go mod`)
- **Database:** MariaDB (platform) + GORM (`gorm.io/gorm`, MySQL driver); SQLite per-project app DB (`gorm.io/driver/sqlite`)
- **Other important dependencies:** `github.com/go-redis/redis/v8` (connector rate limiting), `github.com/robfig/cron/v3` (scheduled tasks), `google.golang.org/api` (Google Play), `github.com/cidertool/asc-go` (Apple App Store Connect), `gopkg.in/gomail.v2` (SMTP), `github.com/gin-contrib/sessions` (sessions)

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `golang/main.go` | Gin bootstrap, middleware, cron startup, listens on configured port |
| `golang/route/api.go` | **Single source of truth** for all HTTP routes (`/v1/...`) |
| `golang/app/controller/` | HTTP handlers |
| `golang/app/service/` | Business logic (calls DB, outbound HTTP, etc.) |
| `golang/app/model/mariadb/` | MariaDB entities / repositories |
| `golang/app/model/sqlite/` | Per-project SQLite models (`client/` vs `server/`) |
| `golang/app/middleware/` | OAuth, API-Key (`API-Key` header), CORS (`auth.json`), sessions |
| `golang/app/helper/config.go` | Loads JSON configs from `{app_root}/config/*.json` |
| `golang/config/*.sample` / `mail.json.example` | Committed templates; real `*.json` ignored by git |
| `golang/database/` | `ConnectMariaDB` / `ConnectSQLite` |
| `golang/storage/` | Templates, tmp areas (e.g. cleanup targets) |
| `service/` | Linux systemd-oriented deployment configs (outside `golang/` app) |
| `docker-compose.yml` | Builds from `golang/` (`context: ./golang`); mounts `golang/config` and `golang/log` |

## 4. Exposed Endpoints

All API routes registered in `route.Api` use the **`/v1` prefix** (no extra global prefix in `main.go`). Optional static files: Gin `router.Static("storage", ...)` when `application.enableLocalStaticHosting` is true (`golang/route/api.go`).

Auth modes (see `golang/route/api.go` and `golang/app/middleware/oauth.go`):

- **OAuth2 Bearer:** `Authorization: Bearer …` (`OAuthTokenFull` / mixed groups)
- **Config API key:** `API-Key` header matched to `auth.json` → `APIKey` (`ConfigAuth`)
- **Either:** OAuth or config API key (`OAuthTokenOrConfigAuth`)
- **None:** Selected routes (e.g. OAuth login, `/v1/callback`, some published-app routes, password reset)
- **Admin / role:** Nested middleware (`AdminRole`, `AllowedRole`) on subsets

Below: **logical groups** (not every method/path variant). For the exhaustive list including handler names, see `golang/route/api.go`.

| Method(s) | Path (under `/v1`) | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/callback` | SSO/OAuth redirect callback | `golang/app/controller/oauth.go` (via OAuth service) | No |
| `GET` | `/oauth/login`, `POST /oauth/logout`, `POST /oauth/tokens/introspect` | OAuth login/out, token introspect | `golang/app/controller/oauth.go` | Mixed |
| `POST` | `/rate-limit` | Connector raw V2 rate limit; validates `api_keys` row (+ optional Redis) | `golang/app/controller/connector_rate_limit.go` | No — `X-API-Key` or `API-Key` header (see `extractConnectorRawAPIKey`) |
| `POST` | `/sso/company/synchronize`, `/sso/user/synchronize`, `/sso/user/.../api-keys` | Subscription/SSO sync, API keys for SSO users | `golang/app/controller/sso.go` | Config API key |
| Various | `/application-control/...`, `GET .../runtime-license-validate-application-control` | Compile/storage/publish pipeline control; runtime license check | `golang/app/controller/application_control.go` | OAuth and/or Config per route |
| Various | `/applications/...` | Apps CRUD, build/publish/deploy, banners, SQLite-backed DB/table REST, icons, AI project create | `golang/app/controller/application.go`, `banner.go`, `database.go`, `table_content.go` | Public / OAuth / Config / Mixed |
| Various | `/project-generator/generate-server` | Generate server project | `golang/app/controller/` | OAuth |
| Various | `/cookies/...`, `POST .../v5-proxy-request` | Stored cookies + HTTP proxy forwarding | `golang/app/controller/cookie.go` | OAuth |
| Various | `/users/...`, `/companies/...` | User/company CRUD, passwords, photo, EMSM listing | `golang/app/controller/user.go`, `company.go` | Mixed + admin gates |
| Various | `/licenses/...`, `/notifications`, `/help`, `/summary` | Licenses, notifications, help, dashboard summary | respective controllers under `golang/app/controller/` | Mostly OAuth |
| Various | `/audits/...`, `/logs/...`, `POST /logs` | Release/build audits, application audit log CRUD | `golang/app/controller/audit.go` | OAuth / Mixed |
| Various | `/build-types`, `/tools/...` | Build types; code-parser import/export; automation | `golang/app/controller/` | Mixed |
| Various | `/support/...`, `/support/admin/...` | User + admin support tickets | `golang/app/controller/support.go` | OAuth + role |
| Various | `/external_apis/...` | External API definitions, domains, functions; business connector helpers | `golang/app/controller/external_api.go` | OAuth / Mixed |
| Various | `/autopilot/...` | AI/autopilot app settings/files/plugin assets | `golang/app/controller/autopilot.go` | OAuth |
| Various | `/plugins/search`, `/plugins/action` | Plugin search/actions via service manager | `golang/app/controller/plugin.go` | OAuth |
| Various | `/supabase/...` | Proxy to Supabase management-style APIs (avoid browser CORS) | `golang/app/controller/supabase_connection.go` | OAuth |
| Various | `/mcp/...`, `/custom_ui/...` | MCP session persistence; custom UI payloads | respective controllers | OAuth |

## 5. Outbound API Calls

Base URLs and keys are overwhelmingly driven by **`golang/config/api.json`** (see sample keys in `golang/config/api.json.sample`). Call sites use `helper.ConfigRead(..., "api")` / `helper.ConfigGet(..., "api", ...)` in services and libraries.

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| SSO / IdP (`api.sso.url` + paths) | Various | Paths like `/oauth/token`, `/api/sso/user`, etc. (from config) | OAuth code exchange, user profile, logout | `golang/app/service/oauth.go`, `golang/app/service/sso.go` |
| License / subscription API (`api.license`) | POST (typical) | Configured paths e.g. `path_subscription`, `path_rtl`, `get_subscription_attr_by_idp` | Subscription enforcement, RTL, cleanup/sync flows | `golang/app/service/oauth.go`, `golang/app/service/application.go`, `golang/app/service/autopilot.go`, `golang/app/service/ai_project_cleanup.go`, `golang/app/service/application_control.go` |
| Frontend / main app (`api.main`) | NEEDS CONFIRMATION | Authorize redirect paths | SSO/login redirects | `golang/app/service/oauth.go` |
| Publisher service (`api.publisher`) | NEEDS CONFIRMATION | e.g. `path_publish` with `applicationCode` placeholder | Publish flow | Publisher library usage (see `golang/app/library/publisher/` and services that invoke it) |
| Compiler (`api.compiler.api`) | POST | Compiler HTTP API plus **`api.compiler.scp`** when defined (**SCP** to compiler host is then part of the same submission) | Submit compile jobs, tmp file handling | `golang/app/library/compiler/compiler.go` |
| Parser (`api.parser.url`) | Various | Parser service REST | Tooling import/export pipeline | `golang/app/service/tool.go` |
| AI backend (`api.ai_backend`) | GET/POST | Deployment-specific routes | Autopilot/settings and related | `golang/app/library/ai/backend.go`, `golang/app/service/autopilot.go` |
| OpenAI / Anthropic (`api.open_ai`, `api.anthropic`) | NEEDS CONFIRMATION | Configured bases + deployments | AI features INFERRED where used | Evidence: keys in `api.json.sample`; trace per-feature with codebase search |
| Supabase OAuth / Management | Various | Supabase-hosted API URLs constructed in service | Org/project/token operations | `golang/app/service/supabase_connection.go` |
| “Platform” socket / builder callbacks (`api.platform.socket_url`, `emobiq_main_backend_url`, `emobiq_v5_connector_url`) | NEEDS CONFIRMATION | WebSocket/long-poll patterns INFERRED | Build/compile progress, connector | `golang/app/library/socket/socket.go`, publisher/build paths in config sample |
| EMSM Service Manager (`api.service_manager.api`) | Various | `/api`-prefixed EMS manager routes | Plugins, `/users/service_manager`, package actions | `golang/app/service/plugin.go`, `golang/app/service/sso.go` |
| Arbitrary origins (cookie proxy, user-defined external APIs) | Proxy | Caller-supplied target URL subject to validation | `v5-proxy-request` behavior | `golang/app/service/cookie_proxy.go` |
| Google Play Developer API | Various | Android Publisher API v3 | Store listing / publish INFERRED | `golang/app/library/publisher/google_play_store.go` |
| Apple App Store Connect API | Various | ASC API via `asc-go` | Store listing / publish INFERRED | `golang/app/library/publisher/apple_app_store.go` |
| SMTP server (`mail.json` / `gomail`) | SEND | Provider-defined | Transactional mail | `golang/app/helper/mail.go` |
| Redis (`database.json` → `redis.connector_raw_rate_limit`) | Redis protocol | Rate limit buckets | Connector rate limiting | `golang/config/database.json.sample` comments; consumer in connector rate-limit path |

## 6. Database / Models / Tables

### MariaDB (platform)

Primary schema via GORM models in `golang/app/model/mariadb/`. Documented examples include **`users`**, **`companies`**, **`applications`**, **`applications_type`**, **`roles`**, **`licenses`**, **`audits`**. Other structs in the same package map to pluralized GORM table names unless `TableName()` overrides (see `applications_type.go`, `support_ticket_activity.go`). Entities include **builds**, **build_types**, **publishings**, **publishing_channels**, **platform**, **oauth_***, **banners**, **cookies**, **external_apis**, **external_api_domains**, **api_keys**, **packages**, **custom_uis**, **mcp_sessions**, **support_ticket***, etc. **Read/Write:** Both via services.

Evidence: model files under `golang/app/model/mariadb/`.

### SQLite (per application)

Project file path pattern from **`database.json`** → `sqlite.database` relative to storage (sample: `data.db`). Models under `golang/app/model/sqlite/client/` (e.g. pages, snippets, custom styles) and `golang/app/model/sqlite/server/` (databases, tables, services, configs, migrations, etc.). Used for IDE-generated app content and build metadata. **Read/Write:** Via `database.ConnectSQLite(applicationCode, session)`.

### Redis

Optional; documented in **`database.json.sample`** for connector rate limiting (not MariaDB tables).

## 7. Jobs / Queues / Cron / Workers

| Name | Type | Purpose | Source file |
|---|---|---|---|
| AI project cleanup | Cron (`0 0 * * *` daily midnight) | Removes stale AI projects; may call license/AI-backend HTTP | `golang/app/service/ai_project_cleanup.go` |
| Cookie cleanup | Cron (`0 0 * * *` daily midnight) | Deletes expired cookie rows | `golang/app/service/cookie_cleanup.go` |
| Tmp file cleanup | Cron (`0 * * * *` hourly) | Clears expired files under `storage/tmp/bcapi` via system `find` | `golang/app/service/tmp_file_cleanup.go` |
| Cron coordinator | Startup | Registers above when enabled in config | `golang/app/service/cron.go`, `golang/main.go` |

Toggle flags: `application.scheduler_settings` in `application.json` / sample (`ai_project_cleanup_enabled`, `cookies_cleanup_enabled`, `tmp_file_cleanup_enabled`).

**Message queues / async build work** are not implemented in this repo; they live in separate **compiler** and **builder** repositories (see those repos for queue/bus details). This service still exposes HTTP callbacks consumed by builders/compilers (`api.platform` URLs in config).

## 8. Configuration & environment

### Primary: JSON files under `{app_root}/config` (path joined in `helper.ConfigGet` / `helper/config.go`)

The running binary resolves **`DirectoryAppRoot()`** + **`/config/<name>.json`**. Templates committed as **`golang/config/*.sample`** and **`golang/config/mail.json.example`**. **`helper.ConfigReader`** wires `auth`, `api`, `application`, `database`, `mail` (`golang/app/helper/config.go`).

**Why primary:** Middleware and services consistently read these files at runtime (`ConfigRead` / `ConfigGet`).

| Key / name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `auth.json`: `allowedOrigins`, `defaultOrigin` | CORS allowlist | `golang/config/auth.json.sample`; `golang/app/middleware/cors.go` |
| `auth.json`: `APIKey` | Backend `API-Key` header auth | `auth.json.sample`; `golang/app/middleware/oauth.go` |
| `application.json`: `port`, `storageTemplatePath`, `projectStoragePath`, token lifetimes, `scheduler_settings`, `enableLocalStaticHosting`, `sessionSecret` | Server port, filesystem roots, JWT/session-related timings, cron toggles, dev static hosting | `application.json.sample`; `golang/main.go`, `golang/route/api.go`, schedulers |
| `database.json`: `mariadb`, `sqlite`, `redis` | DB credentials/paths; optional Redis for rate limits | `database.json.sample`; `golang/database/` |
| `api.json`: `sso`, `license`, `publisher`, `compiler`, `platform`, `service_manager`, `parser`, AI providers, `supabase_credentials`, `ai_backend`, etc. | All major outbound integrations | `api.json.sample`; services under `golang/app/service/`, `golang/app/library/` |
| `mail.json` | SMTP host/port/credentials (`mail.json.example`) | `golang/app/helper/mail.go` |

### Other (secondary)

| Variable / key | Purpose | Used in |
|---|---|---|
| `helper.EnvAppCompiled` / env flags | Paths when `go run` vs compiled binary | `golang/app/helper/environment.go` (and docs in `golang/AGENTS.md`) |
| Deployment mounts | Compose/systemd may mount `/config`, `/log` | `docker-compose.yml` (`golang/config`, `golang/log`), `service/` |

**Rules observed:** Never commit populated `golang/config/*.json` (gitignored—see AGENTS/security notes).

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| MariaDB | Database | Canonical platform data | `golang/database/mariadb.go`, models |
| SQLite files on storage volume | Database | Per-app project databases | `golang/database/sqlite.go`, SQLite models |
| Redis (optional) | Cache / counters | Connector rate limiting | `golang/config/database.json.sample`; rate-limit route |
| SSO / OAuth server | Auth API | Tokens, callbacks, users | `golang/app/service/oauth.go`, `/v1/callback` |
| License / subscription service | API | Entitlements, RTL, subscription payloads | Config `license.*`; multiple services POST JSON |
| Compiler / builder / publisher stack | API + SCP; queues/message buses (**compiler** & **builder** repos) | Builds, binaries, listings | `api.compiler`, `api.platform`, `golang/app/library/compiler`, `golang/app/library/publisher` |
| EMSM / plugins host | API | Plugin catalog and actions | `api.service_manager` |
| AI / parser / autopilot backends | API | Automation and AI-assisted flows | `api.ai_backend`, `api.parser`, `open_ai`, `anthropic` samples |
| Supabase cloud | OAuth + management API | Project/org management via proxy routes | `golang/app/service/supabase_connection.go` |
| Google / Apple publishing APIs | API | Mobile store integrations | Publisher libraries |
| SMTP relay | Mail | Password / notification email | `golang/app/helper/mail.go` |
| Frontends / other Orangekloud services | HTTP clients | Call authenticated main-backend endpoints | SSO redirect targets, builders hitting `/v1/...` per `api.json.sample` |

## 10. Main Flows

### Flow: OAuth login and API access

1. Browser hits `GET /v1/oauth/login` → redirects using `api.sso` + `api.main` settings (`oauth` service/controller).
2. IdP redirects to `GET /v1/callback` with authorization code (`route/api.go`).
3. Backend exchanges token, resolves user/session, persists OAuth tokens (`oauth.go` service + MariaDB oauth models).
4. Subsequent calls use `Authorization: Bearer` through `OAuthTokenFull` middleware.

### Flow: Application build → compile → artifact handoff

1. Client triggers build under `/v1/applications/:application_code/build` (OAuth).
2. Service compresses/syncs sources and submits to the compiler (`golang/app/library/compiler/compiler.go`): HTTP to **`api.compiler`**, and **SCP** to the compiler host when **`api.compiler.scp`** is configured.
3. **Compiler / builder** services (separate repos) run async work backed by **their** message queues; they call back into this API as needed.
4. Builder/compiler callbacks hit **config-backed main-backend URLs** (`api.platform.emobiq_main_backend_url` in sample) e.g. build complete: `POST /v1/applications/:application_code/build/complete` with **config API key** (`route/api.go` `applicationConfigAuth` group).
5. Status/progress may use **platform socket** URL (`golang/app/library/socket/socket.go`) — INFERRED wiring per deployment.

### Flow: Publish / store listing

1. User initiates publish routes under `/v1/applications/.../publish*` (OAuth).
2. Backend coordinates with **publisher** and **Google/Apple** libraries as applicable (`golang/app/library/publisher/`).
3. License/subscription checks may call **license** API (`application`, `application_control`, etc.).

### Flow: SSO user / company sync (subscription system)

1. External subscription service calls `POST /v1/sso/company/synchronize` and `POST /v1/sso/user/synchronize` with **config API key** (`route/api.go`).
2. `SSO` service updates MariaDB company/user records and related data (`golang/app/service/sso.go`).

### Flow: Supabase from browser (CORS bypass)

1. Frontend calls `POST /v1/supabase/...` with user OAuth token.
2. `supabase_connection` service calls Supabase’s HTTP APIs using configured credentials (`golang/app/service/supabase_connection.go`).

## 11. Things Other Repos May Depend On

- **Stable `/v1` REST surface** as registered in `golang/route/api.go` (breaking changes affect frontends, builders, compilers, EMSM, subscription webhooks).
- **OAuth callback URL** `.../v1/callback` must match IdP app registration (`api.json.sample` `sso.redirect_url`).
- **Service-to-service auth** via `API-Key` header matching `auth.json` for build/publish/SSO routes.
- **Published app endpoints** under `/v1/applications/published/...` (public / low-auth consumers).
- **Rate limit endpoint** `POST /v1/rate-limit` for connector raw V2 + optional Redis semantics.
- **Filesystem contract:** `projectStoragePath` and SQLite `data.db` locations must stay consistent across build agents and this API.