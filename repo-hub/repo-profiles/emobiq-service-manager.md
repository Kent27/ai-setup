# Repo Profile: emobiq-service-manager

Last updated: 2026-05-07
Confidence: High

## 1. Purpose

**Primary product focus:** this service backs the **eMOBIQ plugins** capability—cataloging and delivering **plugin packages** (versioned `.tar` archives, metadata, readme/config keys, icons), shared **libraries**, and **developer** accounts, plus **entitlement** checks (`canCreatePlugin` / `canUsePlugin`) from subscription attributes (`es5/api/lib/auth.js`).

Backend and web UI for **eMOBIQ Service Manager (ESM)** more broadly: SSO/browser login, CLI OAuth handoff, and integration hooks for the main eMOBIQ platform (`mainBackendUrl`) and subscription service.

## 2. Tech Stack

- Language: JavaScript (ES5 / CommonJS in `es5/`)
- Framework: Express-like app from vendored **`elasticx`** (`./packages/elasticx.tgz`), entity routers, EJS views
- Runtime: Node.js (`node ./es5/index.js` via `npm run startDev`)
- Package manager: npm (`package-lock.json` v3)
- Database: PostgreSQL (connection string in `application.databaseUrl`)
- Other important dependencies: `passport` + `passport-local`, `express-session`, `axios`, `request` / `request-promise-native`, `multer`, `picomatch`, `ejs`, `bcrypt`, PM2 (`npm start`)

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `es5/index.js` | Process entry: mounts API (`/api`), web (`/`), CORS `GET /sso.js` |
| `es5/api/index.js` | DB entities, auth allowlist (`authEndpoints`), entity REST for package/user/lib, `/api/esm-user-create`, `/api/plugin/access` |
| `es5/api/lib/pkg.js` | Package-related HTTP routes (deps, tags, readme, editor-data, icons, user actions) |
| `es5/api/lib/lib.js` | Library dependency listing routes |
| `es5/api/lib/auth.js` | User sync/create from platform; subscription check helper for plugin access |
| `es5/api/lib/passport.js` | `/api/login`, session, `/api/me`, `/api/register`, admin create/sync user, `/api/sso-logout` |
| `es5/api/models/*.js` | `PgEntity` definitions (table names, aliases, `walk` / `subEnt`) |
| `es5/web/index.js` | EJS pages, `/login`, `/js/auth/config.js` generation, `/docs` file server, `apiRequest` to local `/api` |
| `es5/web/oauthCallback.js` | Browser OAuth callback `/v1/callback-service-manager` |
| `es5/web/cliOauthCallback.js` | CLI OAuth callback `/v1/callback-cli` |
| `es5/web/cliTokenApi.js` | One-shot token retrieval `GET /v1/cli/token` |
| `es5/middleware/ssoTokenMiddleware.js` | Bearer/cookie SSO token → introspection + DB/SSO user hydration |
| `es5/middleware/apiKeyMiddleware.js` | Sets `req.hasValidApiKey` / `req.isAuthenticated` from `x-api-key` |
| `config/config.js` | Local runtime config (gitignored) |
| `config/config.js.sample` | Field checklist / shape reference |
| `archive/` | Package `.tar` artifacts (`archive/package/<id>/<version>.tar` per API conventions) |
| `statics/`, `statics_cors/` | Static assets; `statics_cors/sso.js` for cross-origin script |
| `docs/` | Static documentation files served under web `/docs` |

## 4. Exposed Endpoints

Single HTTP server: API under **`/api`**, web under **`/`**, plus **`/v1/*`** callbacks and **`/struct/*`** object dump routes. Auth for many sensitive `/api/*` paths is enforced by **picomatch patterns** in `es5/api/index.js` (`authEndpoints`): unauthenticated requests to matching paths get `401`.

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/sso.js` | Serve CORS SSO helper script | `es5/index.js` | No |
| Various | `/api/*` | CRUD and query for **package**, **user**, **lib** (via `elApp.registerEntityRouter`) | `es5/api/index.js`, `es5/package/elasticx.js` | Yes for paths under `authEndpoints` patterns if no session/SSO/API-key auth; otherwise varies |
| `POST` | `/api/esm-user-create` | Create/update ESM user from platform payload | `es5/api/index.js`, `es5/api/lib/auth.js` | `x-api-key` must match `platform.serviceManagerApiKey` |
| `GET` | `/api/plugin/access` | Subscription-backed plugin create/use flags | `es5/api/index.js`, `es5/api/lib/auth.js` | `ssoUserMiddleware` + `req.user` |
| `POST` | `/api/login`, `GET` `/api/logout`, `POST` `/api/sso-logout`, `GET` `/api/me`, `POST` `/api/register`, `POST` `/api/create-user`, `POST` `/api/sync-user` | Passport-local and user admin flows | `es5/api/lib/passport.js`, `es5/api/lib/auth.js` | Varies (local login vs admin-only vs token for SSO logout) |
| `GET` | `/api/package/:id/deps`, `/api/package/:id/libs`, `/api/packagelibs`, … | Package graph, tags, readme, editor payload, icon | `es5/api/lib/pkg.js` | Matched patterns require auth (see `authEndpoints`) |
| `GET` | `/api/lib/:id/deps`, `/api/libdeps` | Library dependencies | `es5/api/lib/lib.js` | Pattern-based (lib deps paths in `authEndpoints`) |
| `GET` | `/api/archive/**` | Static files from `archive/` | `es5/api/index.js` | Yes (pattern) |
| `GET` | `/struct/package`, `/struct/user`, `/struct/lib` | Returns registered model module objects (structure dump) | `es5/api/index.js`, `elasticx.registerObjectRouter` | No (not in `authEndpoints`) |
| `GET` | `/` | Home EJS | `es5/web/index.js` | No |
| `GET` | `/login` | Redirect to SSO authorize URL | `es5/web/index.js` | No |
| `GET` | `/js/auth/config.js` | Runtime-filled auth config from template | `es5/web/index.js`, `statics/js/auth/config-template.js` | No |
| `GET` | `/user`, `/user/:userId` | Developer list (admin) / self profile | `es5/web/index.js` | Admin / self (application logic) |
| `GET` | `/about-us`, `/docs/*` | Static/marketing/docs | `es5/web/index.js` | No |
| `GET` | `/v1/callback-service-manager` | OAuth code exchange (browser) | `es5/web/oauthCallback.js` | No (uses code + SSO) |
| `GET` | `/v1/callback-cli` | OAuth code exchange (CLI) | `es5/web/cliOauthCallback.js` | No |
| `GET` | `/v1/cli/token` | One-time CLI token by `sessionId` | `es5/web/cliTokenApi.js` | No |

Exact verbs/paths for generic entity CRUD follow `elasticx` `registerEntityRouter` behavior on `/api/package`, `/api/user`, `/api/lib`.

## 5. Outbound API Calls

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| Main backend (`platform.mainBackendUrl`) | `POST` | `/oauth/tokens/introspect` | Validate Bearer token for API/web requests | `es5/middleware/ssoTokenMiddleware.js` |
| Main backend (`platform.mainBackendUrl`) | `GET` | `{pathUserServiceManager}?user_id=...` (e.g. `/users/service_manager`) | Fetch user from platform for register/sync | `es5/api/lib/auth.js` |
| Main backend (`platform.mainBackendUrl`) | `POST` | `/oauth/logout` | Revoke SSO session server-side | `es5/api/lib/passport.js` |
| SSO (`platform.ssoUrl`) | `POST` | `/oauth/token` | Authorization code → access token (browser + CLI) | `es5/web/oauthCallback.js`, `es5/web/cliOauthCallback.js` |
| SSO (`platform.ssoUrl`) | `GET` | `/api/user` | Profile when ESM DB has no row for SSO subject | `es5/middleware/ssoTokenMiddleware.js` |
| Subscription service (`platform.subscriptionUrl`) | `POST` | `/api/open/get_subscription_attr_by_idp` | Plugin create/use entitlement | `es5/api/lib/auth.js` |
| Same-origin Service Manager | `GET` | `apiURL + '<path>'` e.g. `/api/user` | Server-side JSON for EJS pages | `es5/web/index.js` (`request` module) |

Browser login **redirects** users to `{ssoUrl}/oauth/authorize?...` (not a server-side HTTP client call).

## 6. Database / Models / Tables

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| `esm_package` | Package metadata (`p`) | Read / Write | `es5/api/models/package.js` |
| `esm_user` | Developers / roles (`u`) | Read / Write | `es5/api/models/user.js` |
| `esm_lib` | Shared libraries (`l`) | Read / Write | `es5/api/models/lib.js` |
| `esm_lib_deps` | Library dependencies (`ld`) | Read / Write | `es5/api/models/lib_deps.js` |
| `esm_package_contribs` | Package contributors (`pc`) | Read / Write | `es5/api/models/package_contribs.js` |
| `esm_package_tags` (`pt` alias in code; named opposite import in `index.js`) | Package tags | Read / Write | `es5/api/models/package_tags.js` |
| `esm_package_deps` (`pd` alias) | Package dependencies | Read / Write | `es5/api/models/package_deps.js` |
| `esm_package_libdeps` | Package–library deps (`pl`) | Read / Write | `es5/api/models/package_libdeps.js` |
| `esm_package_editlib` | Edit-time lib links (`el`) | Read / Write | `es5/api/models/package_editlib.js` |
| `user_plugin_actions` | Per-user plugin actions (`ua`) | Read / Write | `es5/api/models/user_plugin_actions.js` |
| `esm_package_plugins` | Model file present | NEEDS CONFIRMATION (not registered in `entityModels` in `es5/api/index.js`) | `es5/api/models/package_plugins.js` |

Raw SQL also used in `ssoTokenMiddleware` against `esm_user` for `external_sso_user_id`.

## 7. Jobs / Queues / Cron / Workers

No jobs, queues, cron tasks, or workers were found.

Process management: `npm start` runs **PM2** with watch on `es5/index.js` (not a scheduled job).

## 8. Configuration & Environment

### Primary: `config/config.js` (CommonJS export, local-only)

Runtime reads **`config/config.js`** via `require` across API, web, and middleware (`config/.gitignore` keeps secrets out of git). There is **no** committed `.env` pipeline; integration URLs and keys are **Node module exports**.

| Key / Name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `application.port` | HTTP listen port | `config/config.js.sample`, `es5/index.js` |
| `application.databaseUrl` | PostgreSQL connection string tail (used as `postgres://` + value) | `config/config.js.sample`, `es5/api/index.js`, `es5/middleware/ssoTokenMiddleware.js` |
| `application.cliCommand` | Injected into `/docs` as `%CLI_COMMAND%` | `config/config.js.sample`, `es5/web/index.js` |
| `platform.serviceManagerUrl` | Base for OAuth `redirect_uri` (`.../callback-service-manager`, `.../callback-cli`) | `config/config.js.sample`, `es5/web/oauthCallback.js`, `es5/web/cliOauthCallback.js` |
| `platform.serviceManagerTokenKey` | Cookie name for SSO access token (default `_em_id`) | `config/config.js.sample`, `es5/middleware/ssoTokenMiddleware.js`, OAuth callbacks |
| `platform.serviceManagerApiKey` | Validates `x-api-key` for `/api/esm-user-create` | `config/config.js.sample`, `es5/api/lib/auth.js` |
| `platform.ssoUrl`, `ssoClientId`, `ssoClientSecret` | Web SSO authorize + token | `config/config.js.sample`, `es5/web/index.js`, `es5/web/oauthCallback.js` |
| `platform.ssoCLIClientId`, `ssoCLIClientSecret` | CLI OAuth client | `config/config.js.sample`, `es5/web/cliOauthCallback.js` |
| `platform.mainBackendUrl` | Introspect, platform user API, logout | `config/config.js.sample`, `es5/middleware/ssoTokenMiddleware.js`, `es5/api/lib/auth.js`, `es5/api/lib/passport.js` |
| `platform.apiKey` | Header `API-Key` to main backend user paths | `config/config.js.sample`, `es5/api/lib/auth.js` |
| `platform.pathUserServiceManager` | Path under main backend for user lookup | `config/config.js.sample`, `es5/api/lib/auth.js` |
| `platform.subscriptionUrl` | Subscription service origin for plugin access | `config/config.js.sample`, `es5/api/lib/auth.js` |
| `platform.oauthClientBootstrapSecret` | Optional post-login OAuth client bootstrap chain | `config/config.js.sample`, `es5/web/oauthCallback.js` |
| `platform.mainFrontendUrl`, `clientUrl`, etc. | Shown in templates / `config.js` | `config/config.js.sample`, `es5/web/index.js` |

### Other / Secondary

| Variable / Key | Purpose | Used In |
|---|---|---|
| Docker / compose | Sample deployment layout | `Dockerfile`, `docker-compose.yml.sample` (sample only) |

**Note:** `config/config.js.sample` is a **hand-maintained checklist** and may contain syntax/formatting issues; treat `config.js` as the real contract.

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| PostgreSQL | Database | Package, user, lib, and relation tables | `es5/api/index.js`, models |
| Main eMOBIQ backend | API | Token introspection, user provisioning data, OAuth logout | `es5/middleware/ssoTokenMiddleware.js`, `es5/api/lib/auth.js`, `es5/api/lib/passport.js` |
| SSO / IdP (`ssoUrl`) | Auth | OAuth2 authorize + token + user profile | `es5/web/*.js`, `es5/middleware/ssoTokenMiddleware.js` |
| Subscription / entitlement service | API | Plugin create/use flags | `es5/api/lib/auth.js` |
| eMOBIQ clients (browser, CLI, other services) | Consumers | Call Service Manager APIs and OAuth callbacks | `es5/api/index.js`, `es5/web/*` |

## 10. Main Flows

### Flow: Browser SSO login

1. User hits `/login`; server redirects to `{ssoUrl}/oauth/authorize` with `redirect_uri` derived from `platform.serviceManagerUrl` and `state` carrying safe `returnUrl`. (`es5/web/index.js`)
2. SSO returns to `GET /v1/callback-service-manager` with `code`. (`es5/web/oauthCallback.js`)
3. Server exchanges `code` at `{ssoUrl}/oauth/token`, sets cookie (`serviceManagerTokenKey`). (`es5/web/oauthCallback.js`)
4. Subsequent API requests send cookie or `Authorization: Bearer`; middleware introspects via `{mainBackendUrl}/oauth/tokens/introspect` and loads `esm_user` or SSO profile. (`es5/middleware/ssoTokenMiddleware.js`)

### Flow: Plugin package publish / metadata update

1. Authenticated client calls entity router endpoints under `/api/package` with multipart fields (`pack`, optional `icon`) per API conventions. (`es5/api/index.js`, `es5/api/AGENTS.md`)
2. Middleware validates roles for destructive operations; versions must increase (`es5/api/lib/compare.js` referenced in AGENTS).
3. Archive stored under `archive/package/...`; metadata in `esm_package` and related tables.

### Flow: Plugin access (entitlement)

1. Client calls `GET /api/plugin/access` with an authenticated user (SSO subject on `req.user`). (`es5/api/index.js`, `es5/api/lib/auth.js`)
2. Server posts to `{subscriptionUrl}/api/open/get_subscription_attr_by_idp` and returns `canCreatePlugin` / `canUsePlugin`. (`es5/api/lib/auth.js`)
3. Per-user plugin actions (e.g. bookmark/add) can be recorded via `POST /api/user/action` in `es5/api/lib/pkg.js` (backed by `user_plugin_actions`).

### Flow: Platform-provisioned ESM user

1. External service `POST /api/esm-user-create` with `x-api-key` matching `serviceManagerApiKey`. (`es5/api/lib/auth.js`)
2. User row inserted or updated in `esm_user`.

### Flow: CLI OAuth handoff

1. User completes SSO with CLI client; callback `GET /v1/callback-cli` exchanges code, stores token in memory keyed by `sessionId`, redirects to caller. (`es5/web/cliOauthCallback.js`)
2. CLI retrieves token once via `GET /v1/cli/token?sessionId=...`. (`es5/web/cliTokenApi.js`)

## 11. Things Other Repos Depend On

- **Plugin surface**: `/api/plugin/access` (entitlement), `/api/package` + archive paths (plugin artifacts), `/api/user/action` (user–plugin actions), and related routes in `pkg.js` / `lib.js`.
- **Stable HTTP routes** (broader ESM): `/api/package`, `/api/lib`, `/api/user` (entity API), custom `/api/*` in `pkg.js` / `lib.js`, `/api/esm-user-create`, `/struct/*` model exports.
- **OAuth redirect URIs**: `{serviceManagerUrl}/callback-service-manager` and `{serviceManagerUrl}/callback-cli` must match SSO app registration. (`es5/web/oauthCallback.js`, `es5/web/cliOauthCallback.js`)
- **Cookie / header contract**: Token cookie name from `platform.serviceManagerTokenKey` (default `_em_id`); Bearer alternative for APIs. (`es5/middleware/ssoTokenMiddleware.js`)
- **`GET /js/auth/config.js`**: Browser clients expect runtime SSO URLs and client id from substituted template. (`es5/web/index.js`)
- **`x-api-key`** on `/api/esm-user-create` aligned with `platform.serviceManagerApiKey`.
- **Archive layout**: `archive/package/<id>/<version>.tar` and `icons/logo.png` conventions consumed by package flows.
- **Main backend + SSO URLs** for introspection, user sync paths, and token exchange must stay compatible with this codebase.

## 12. Unknowns / Needs Confirmation

- NEEDS CONFIRMATION: Full matrix of **which** `/api/*` entity methods are public vs protected (only explicit `authEndpoints` globs enforce 401; other routes may be reachable without session).
- NEEDS CONFIRMATION: Whether `esm_package_plugins` / `es5/api/models/package_plugins.js` is legacy or used outside `entityModels`.
- NEEDS CONFIRMATION: Production deployment topology (reverse proxy base path, TLS termination) relative to `serviceManagerUrl` and `redirect_uri` composition.
