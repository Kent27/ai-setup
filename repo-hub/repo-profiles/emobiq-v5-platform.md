# Repo Profile: eMOBIQ V5 Platform

Last updated: 2026-05-07  
Confidence: Medium

## 1. Purpose

Multi-component low-code platform for building and running mobile apps: visual editor (`edit/`), Cordova-oriented runtime (`www3/`), REST API (`api/`), service connectors (`connector/`), mobile build tooling (`builder/`), shared PHP core (`lib/`), and a Node WebSocket helper (`econnector/`). Documentation references a separate **Q Framework** clone used as the PHP ORM/runtime dependency.

**Operational note:** In current deployments this repo also acts as an **HTTP proxy** for **autopilot-debug-agent**, which sends outbound HTTP through **`/connector/raw/`**. That use is one of the **main reasons `connector/raw/` runs in production**—distinct from eMOBIQ AI features (those calls are not “for eMOBIQ AI”).

## 2. Tech Stack

| | |
|---|---|
| **Language** | PHP (5.6 legacy / 8.2 for newer parts per project docs), JavaScript |
| **Framework** | Custom MVC in `lib/core/` (`core\App`, `core\Controller`, `core\Request`); Q Framework entities in `lib/entity.php` |
| **Runtime** | Apache HTTP Server; Node.js for `econnector/` and parts of `builder/` |
| **Package manager** | No root `composer.json` / `package.json`; isolated Composer under some connectors (e.g. `connector/paypal/`, `crmconnector/`); large `node_modules` under `builder/ios/` |
| **Database** | MariaDB (primary platform DB per README/AGENTS); SQLite per-app `data.db` paths in API config sample |
| **Other** | jQuery 4, Cordova 12, Socket.IO (`econnector/`), Bitbucket Pipelines deploy (`bitbucket-pipelines.yml`) |

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `AGENTS.md` | High-level platform map and conventions |
| `README.md` | Local setup (Apache, PHP, MariaDB, Node, Cordova, JDK, etc.) |
| `lib/` | Shared PHP core (`core/`), entities (`entity.php`), helpers, `common/Models/` |
| `api/` | JSON REST API entry `index.php`, controllers, middleware, `config/config.php` (from sample) |
| `edit/` | Visual editor (PHP + JS); `config.js`, `lib/config.php` |
| `www3/` | App runtime preview/player; `config.js`, `lib/config.php` |
| `connector/` | ERP/SOAP/REST/paypal/etc. connector endpoints; **`connector/raw/`** is heavily used as an HTTP proxy (including for **autopilot-debug-agent**) |
| `builder/` | Mobile build pipelines (incl. Cordova/iOS tooling) |
| `econnector/` | Express + Socket.IO server (`server.js`, `lib/config.js`) |
| `module/` | Legacy API modules (referenced in AGENTS) |
| `bitbucket-pipelines.yml` | Deploy to dev via SSH `git pull` (no test stage) |
| `api/eMOBIQ Platform.postman_collection.json` | API documentation / manual testing |

## 4. Exposed Endpoints

The platform is deployed behind Apache; exact URL prefixes depend on vhost/docroot (see README). Below lists **application-level** surfaces found in code.

### REST API (`api/index.php`)

Routing uses query parameters **`controller`** and **`action`** (see `lib/core/Request.php::resolve()`). HTTP verb selects `actionActionGet` / `actionActionPost` / … or `actionAction` when no verb suffix exists (`lib/core/Controller.php::run()`).

| Method | Pattern | Purpose | Main handler | Auth |
|---|---|---|---|---|
| GET | `?controller=app&action=index` | Fetch app metadata (+ optional page via `with`) | `api/controller/AppController.php` | Unknown |
| POST | `?controller=app&action=index` | Update app theme; touches archived `app.json` via storage URL | `api/controller/AppController.php` | Unknown |
| POST | `?controller=compile&action=index` | Compile pipeline (tmp assets, SCP tarball, call compiler HTTP API) | `api/controller/CompileController.php` | Unknown |
| POST | `?controller=build&action=complete` | Build completion handling | `api/controller/BuildController.php` | Yes — `API-Key` header vs `platform.api_key` (`middleware/ValidateApiKey.php`) |
| POST | `?controller=build&action=clear` | Build cleanup | `api/controller/BuildController.php` | Yes — same |
| GET | `?controller=build&action=status` | Build status | `api/controller/BuildController.php` | Unknown |
| POST | `?controller=logger&action=index` | Persist audit/log payload | `api/controller/LoggerController.php` | Yes — `ValidateApiKey` on `index` |
| GET | `?controller=logger&action=log` | Query logs for app | `api/controller/LoggerController.php` | Unknown |
| GET | `?controller=packages&action=index` | List packages / asset URIs for app | `api/controller/PackagesController.php` | Unknown |
| GET | `?controller=page&action=index` | Read SQLite pages (`eas_page`) | `api/controller/PageController.php` | Unknown |
| POST | `?controller=page&action=set` | Write/update page | `api/controller/PageController.php` | Unknown |
| POST | `?controller=page&action=delete` | Delete page | `api/controller/PageController.php` | Unknown |
| * | `?controller=record&action=query` … | CRUD against staging SQLite by table name | `api/controller/RecordController.php` | Unknown |

**Other HTTP surfaces**

| Surface | Path / pattern | Purpose | Evidence |
|---|---|---|---|
| Connectors | **`/connector/raw/`** (and v2: `/connector/raw/v2/` — see `connector/raw/v2/index.php`) | Generic REST proxy/integration; **primary ingress for autopilot-debug-agent HTTP calls** (not eMOBIQ AI) | `connector/raw/index.php`, `connector/raw/v2/` |
| Web + editor | `/edit/`, `/www3/` (typical; depends on Apache) | Editor UI and runtime shell | `edit/index.php`, `www3/index.php` |
| econnector | HTTPS listener **12043** (hard-coded); mounts `/api`, `/crm`; Socket.IO | Real-time / CRM-related sockets | `econnector/server.js` |

## 5. Outbound API Calls

| Target | Method | Endpoint / mechanism | Purpose | Source file |
|---|---|---|---|---|
| Compiler service | POST | `{compiler.api}/compile` | Submit compile job with JSON body + API keys | `api/controller/CompileController.php` (`http_post`, headers `API-Key`, `API-Key-Platform`) |
| Socket / notifier | POST | `{platform.socket_url}/broadcast` | Notify clients (e.g. build events) | `api/controller/BuildController.php` |
| SCP/SFTP | INFERRED | Upload tarball / artifacts via `compiler.scp`, `platform.scp` | Ship build inputs/outputs between hosts | `api/controller/CompileController.php`, `api/lib/compile/Tmp.php` |
| Connector hosts | GET/POST etc. | Paths under `connectorURL` / `connectorV2URL` from `www3/config.js.sample` | Runtime talks to nav/raw/sql/mail/etc.; **`rawservice`** maps to generic raw connector | `www3/config.js.sample` (`navservice`, `rawservice`, …) |
| Downstream HTTP APIs | INFERRED | Resolved inside **`connector/raw/`** per request/config | **autopilot-debug-agent** primarily reaches external APIs **via this repo**: inbound **`/connector/raw/`**, then connector forwards outbound HTTP (same mechanism used by runtime `rawservice`) | §1, §10; `connector/raw/` |
| Push / builder (editor PHP) | HTTPS | `_K_PUSH_NOTIFICATION`, `_K_BUILDER_IOS` hosts | Push and iOS builder endpoints | `edit/lib/config.php.sample` |
| Main backend | NEEDS CONFIRMATION | `mainBackendPath` / `_K_URL` patterns | Legacy/core backend usage | `lib/config.php.sample` |

Internal HTTP helper: `lib/core/Http/Client.php` (curl). Various legacy `curl_*` usage elsewhere (e.g. `lib/beessoconnect.php`).

## 6. Database / Models / Tables

| Table / store | Purpose | Read/Write | Source file |
|---|---|---|---|
| `applications` | App records (`application_code`, theme, etc.) | Read/Write | `lib/common/Models/Mariadb/App.php` |
| `users` | Accounts / login fields | Read (joins from App) | `lib/common/Models/Mariadb/User.php`, `lib/entity.php` (`dvec_user`) |
| `packages` | App-linked packages | Read | `lib/common/Models/Mariadb/Package.php` |
| `dv_*` / `users` / … | Legacy Q Framework entities | Mixed | `lib/entity.php` (`dvec_package`, `dvec_app`, …) |
| SQLite `eas_page` | Page definitions per app DB | Read/Write | `lib/common/Models/Sqlite/Page.php`; API staging via `api/lib/StagingDatabase.php` (dynamic table access in `RecordController`) |
| Per-app `data.db` | App SQLite file | Read/Write | `api/config/config.php.sample` (`db.sqlite.database` path pattern) |

## 7. Jobs / Queues / Cron / Workers

No first-party cron definitions, queue workers, or migration runners were found in-repo (aside from third-party `cron` package under `builder/ios/node_modules` and builder queue references like `builder/ad/developer.php` using `bdad_queue`). CI only deploys via SSH.

**Summary:** No jobs, queues, cron tasks, or workers were found as a documented platform subsystem.

## 8. Configuration & environment

This repo is **not** `.env`-first. Integration settings live in **PHP `config.php` and JS `config.js` files** copied from committed **`.sample` templates** (actual configs are gitignored per AGENTS).

### Primary: `*.sample` → local `config.php` / `config.js` (Apache-deployed components)

Values are read via PHP defines (`lib/`, `edit/lib/`, `www3/lib/`) or returned arrays merged in `api/index.php` (`api/config/config.php`). The API layer uses a `config()` helper against that merged array (`api/controller/*`, `middleware/ValidateApiKey.php`).

| Key / area | Purpose (no secret values) | Evidence |
|---|---|---|
| DB host/name/user/password | MariaDB + legacy pgsql key in API array | `lib/config.php.sample`, `api/config/config.php.sample` |
| `compiler.api`, `compiler.api_key`, `compiler.scp.*`, `compiler.test` | Compiler URL, auth, SCP target, dry-run flag | `api/config/config.php.sample`; consumer `api/controller/CompileController.php` |
| `platform.api_key` | Validates `API-Key` header on selected controller actions | `api/config/config.php.sample`; `api/middleware/ValidateApiKey.php` |
| `platform.socket_url`, `platform.socket_server_api_key`, `platform.socket_broadcast_channel` | Socket broadcast integration | `api/config/config.php.sample`; `api/controller/BuildController.php` |
| `platform.emobiq_storage_url`, `platform.emobiq_storage_path`, `platform.scp`, `platform.keystore.*` | Storage URLs, artifact paths, signing keystores | `api/config/config.php.sample` |
| `platformURL`, `connectorURL`, `storageURL`, `mainBackendURL`, service URLs | Runtime/editor endpoints | `www3/config.js.sample`, `edit/config.js.sample` |
| `_K_*` defines | Platform, SSO, chat URL, mail templates, storage | `lib/config.php.sample`, `edit/lib/config.php.sample` |

### Other (secondary)

| Variable / key | Purpose | Used in |
|---|---|---|
| `econnector/lib/config.js` `port` | Node listen port (HTTP stack also binds 12043 in `server.js`) | `econnector/server.js` |
| Per-connector `config/config.php` | External ERP/API endpoints | `connector/*/config/` (pattern in `connector/AGENTS.md`) |

**Rules observed:** Do not commit real `config.php` / `config.js`. Samples may contain placeholders; treat any legacy plaintext passwords in samples as **needs rotation** — do not copy into docs or logs.

## 9. Service Dependencies

| Dependency | Type | Why needed | Evidence |
|---|---|---|---|
| MariaDB | Database | Platform metadata, users, apps | README, models under `lib/common/Models/Mariadb/` |
| SQLite files (`data.db`) | Database | Per-app design-time/runtime data | `api/config/config.php.sample` |
| Q Framework (`../q/`) | Library | ORM / platform PHP stack | `AGENTS.md`, README |
| Compiler backend | HTTP + SCP | Builds mobile artifacts | `CompileController.php`, config sample |
| Socket / broadcast service | HTTP | Push build/status updates | `BuildController.php` |
| econnector / Socket.IO | WebSocket | Editor/runtime realtime (ports from `edit/config.js.sample`) | `edit/config.js.sample`, `econnector/` |
| External ERP/SaaS | HTTP | Via connector scripts | `connector/`, `www3/config.js.sample` |
| **autopilot-debug-agent** | HTTP client | Proxied outbound HTTP through **`/connector/raw/`** | §1; `connector/raw/` |
| Android/iOS toolchain | Build | Cordova / Gradle / Xcode | README, `builder/` |

## 10. Main Flows

### Flow: Edit app in visual editor

1. User loads `edit/` (Apache-served PHP + JS).
2. Editor uses `edit/config.js` / `edit/lib/config.php` for platform, websocket, and connector base URLs.
3. Screen/page data is persisted (paths involving storage URLs — **NEEDS CONFIRMATION** for exact persistence API vs direct DB).

### Flow: Compile app (API)

1. Client POSTs to `compile` controller with `appid`, `compiler_type`, optional release fields (`CompileController::validate()`).
2. API gathers SQLite models (`Page`, `Snippet`, `CustomStyle`, etc.), writes temp workspace, may SCP tarball using `compiler.scp`.
3. API POSTs JSON to `{compiler.api}/compile` with compiler + platform API keys.
4. Completion/cleanup may interact with storage paths and socket broadcast (`BuildController`).

### Flow: Run/preview app (`www3`)

1. Runtime loads `www3/index.php` and `www3/config.js`-driven service URLs.
2. `www3/script/bin/emobiqApp.js` drives UI, events, and connector calls (nav/raw/sql/mail CRM paths from config).

### Flow: HTTP proxy for autopilot-debug-agent (`/connector/raw/`)

1. **autopilot-debug-agent** sends HTTP **to this deployment’s** **`/connector/raw/`** (ingress proxy endpoint).
2. **`connector/raw/`** applies validation/rate limiting where configured (see e.g. `connector/raw/v2/RateLimiter.php` for v2) and **forwards** the request to the **downstream** HTTP API defined by that call’s parameters/config.
3. The downstream response is returned to the agent. **Operating this endpoint reliably is a main production concern** for `connector/raw/`, separate from eMOBIQ AI.

## 11. Things Other Repos May Depend On

- **Stable API contract:** Query-string routing (`controller`, `action`) and JSON responses from `api/index.php`.
- **Compiler integration:** POST body and headers expected by `{compiler.api}/compile` (see `CompileController.php`).
- **Platform API key:** `API-Key` header matching `platform.api_key` for logger/build actions that use `ValidateApiKey`.
- **Connector URL paths:** Paths such as `/connector/raw/`, `/connector/nav/`, etc., referenced from `www3/config.js.sample`.
- **autopilot-debug-agent:** Depends on a stable **`/connector/raw/`** (or equivalent raw connector base URL) on this deployment **as its HTTP proxy** to external services (documented here as ops context; not referenced by name inside this repo’s source tree).
- **Postman collection:** `api/eMOBIQ Platform.postman_collection.json` as external contract documentation.
- **INFERRED:** Mobile apps built via Cordova embed runtime behavior compatible with `www3` scripting conventions.

## 12. Unknowns / Needs Confirmation

- Exact public URL paths for `api/` (rewrite rules vs `api/index.php`) — depends on Apache vhost.
- Whether `compile`, `app`, `page`, `record`, `packages`, and unauthenticated `logger` / `build` actions are protected by network rules, session auth, or other middleware not found under `api/middleware/`.
- Full inventory of outbound HTTP calls from `edit/`, `www3/script/`, and `module/` (large JS surfaces).
- Whether **PostgreSQL** (`pgsql` key in `api/config/config.php.sample`) is still used anywhere or legacy-only.
- **INFERRED:** Separate repos (`emobiq-v3-framework`, compiler service, documentation repo) — referenced in AGENTS/README but not vendored here.
- **autopilot-debug-agent** does not appear as a string in this repository; the dependency description above is **team/deployment context**. Exact agent URL, auth headers, and contract should be confirmed in that repo’s config/docs.
