# Login Flow

## Purpose

Describe how **end users** (and, where documented, **admins**) authenticate into Orangekloud / eMOBIQ surfaces—especially **emobiq-main-frontend** and **orangekloud-subscription**—using **emobiq-main-backend** as the platform OAuth entry and **orangekloud-sso** as the identity provider (Passport OAuth2, web login, and token cookie).

**What triggers the flow**

- **Main platform SPA:** User hits a protected route without a usable session/token, chooses **login**, or lands on **`/login/authorize`** after an IdP redirect.
- **Subscription web:** User opens **`GET /sso/login`** (or follows a link there) for the Laravel session + SSO authorization-code flow.
- **Optional:** **`oauth_chain`** bootstrap on **orangekloud-sso** sends the browser through configured URLs (may include **main-backend**, **subscription**, **service-manager**) when enabled.

## Known Context

- **User-provided:** Login may start from **main-frontend** or **subscription**; covers **normal users** with emphasis on typical app login; **admin** behavior is only documented where repo profiles state it explicitly.
- **Assumption:** Deployments align **callback URLs**, **client IDs**, and **`_em_id` / Bearer cookie** behavior across apps on the SSO domain—**NEEDS CONFIRMATION** per environment.

## Related Docs

- Service map: `repo-hub/maps/service-map.md`
- Database map: `repo-hub/maps/database-map.md`

## Repositories Involved

### `emobiq-main-frontend`

- Repo profile: `repo-hub/repo-profiles/emobiq-main-frontend.md`
- Responsibilities: Browser SPA; **does not** implement OAuth server-side—redirects to **main-backend** for login URL; handles **`/login`** and **`/login/authorize`**; sends API calls with **Bearer from cookies**; on **401**, clears user and returns toward login; uses **`REACT_APP_SSO_URL`** for **logout** navigation (`_em_id` clearance per profile).
- Key files: `frontend/src/common/helper/Redirect.ts` (`getOAuthLoginUrl`), `frontend/src/common/helper/HttpRequest.ts`, `frontend/src/Route.tsx`, `frontend/src/page/` login routes, `frontend/src/modals/Logout.tsx`.

### `emobiq-main-backend`

- Repo profile: `repo-hub/repo-profiles/emobiq-main-backend.md`
- Responsibilities: Central **`/v1`** API; **starts OAuth** via `GET /v1/oauth/login` (redirect using **`api.sso`** + **`api.main`** from config); **receives authorization code** at `GET /v1/callback`; **exchanges token** and **persists OAuth tokens** in MariaDB; exposes **`POST /v1/oauth/logout`**, **`POST /v1/oauth/tokens/introspect`** (used after login by API consumers). **OAuth callback URL** must match IdP registration per profile.
- Key files: `golang/route/api.go`, `golang/app/controller/oauth.go`, `golang/app/service/oauth.go`, `golang/config/api.json` (`sso`, `main`, etc.).

### `orangekloud-subscription`

- Repo profile: `repo-hub/repo-profiles/orangekloud-subscription.md`
- Responsibilities: Laravel app with **web SSO login**: **`GET /sso/login`** redirects to SSO authorize URL (paths from **`settings.sso`** JSON in DB); **`GET /callback`** exchanges **code** at SSO token endpoint; establishes **session**; **`SsoTokenToSession`** middleware aligns **`_em_id`** cookie with **main-frontend / service-manager** (per comment in profile). **`OAUTH_CLIENT_BOOTSTRAP_SECRET`** validates **`oauth_chain`** query on SSO login when present. Optional **main-backend sync** after signup/checkout is **orthogonal** to the minimal login path but uses the same identity (`idp_id`).
- Key files: `app/Http/Controllers/SSO/AuthController.php`, `app/Http/Middleware/SsoTokenToSession.php`, `routes/web.php`, `config/services.php`, admin **`SettingsController`** for `settings.sso`.

### `orangekloud-sso`

- Repo profile: `repo-hub/repo-profiles/orangekloud-sso.md`
- Responsibilities: **Laravel Passport** IdP: **`/oauth/authorize`**, **`POST /oauth/token`**; **web** email/password and **Google** login; **`POST /api/sso/login`** (and related trusted-client JSON APIs); after browser login, may set **`_em_id`** (Bearer cookie) and redirect to **`{MAIN_FRONTEND_URL}/login/authorize?token=&state=`** for SPA adoption; optional **`oauth_chain`** HMAC bootstrap across **main-backend → subscription SSO path → service-manager** when enabled. **Admin Blade UI** exists, but **production admins sign in via orangekloud-subscription**, not SSO **`/admin/login`** (those routes **commented out** per profile).
- Key files: `app/Http/Controllers/Auth/LoginController.php`, `app/Http/Controllers/Clients/ClientsController.php`, `app/Support/OAuthClientBootstrapChain.php`, `app/Support/SubscriptionEmailVerificationRedirect.php`, `config/app.php`, `config/auth.php`.

### `emobiq-service-manager`

- Repo profile: `repo-hub/repo-profiles/emobiq-service-manager.md`
- Responsibilities: **Not** the default entry for **main-frontend** login; involved when **`SERVICE_MANAGER_LOGIN_URL`** is used in **OAuth bootstrap chain** (`orangekloud-sso`), or when users open the **service-manager web** **`/login`** (redirect toward SSO authorize + **`/v1/callback-service-manager`** OAuth callback—separate path from **`/v1/callback`** on main-backend).
- Key files: `es5/web/index.js` (`/login`), `es5/web/oauthCallback.js`, `es5/api/lib/passport.js`, `es5/middleware/ssoTokenMiddleware.js`.

## End-to-End Flow

### A) Typical **main-frontend** user (OAuth via main-backend)

1. User opens a **protected** route in **emobiq-main-frontend** without a valid session cookie → guard sends user toward **`/login`**.
2. **`RedirectToSsoLogin`** / **`getOAuthLoginUrl()`** builds a URL to **emobiq-main-backend** using **`REACT_APP_MAIN_BACKEND_URL`**, **`REACT_APP_MAIN_BACKEND_VERSION`**, and **`REACT_APP_MAIN_BACKEND_LOGIN_PATH`** (exact path segments per frontend profile).
3. Browser requests **main-backend** **`GET /v1/oauth/login`** → backend **redirects** to **SSO / IdP** using **`api.sso`** and **`api.main`** from **`golang/config/api.json`** (`oauth` service).
4. User completes authentication at **orangekloud-sso** (authorization code or web login flow as presented by SSO/Passport).
5. IdP redirects browser to **main-backend** **`GET /v1/callback`** with **authorization `code`** (and typical OAuth query parameters).
6. **emobiq-main-backend** **`oauth`** service **exchanges the code** with SSO (`api.sso`-configured paths such as **`/oauth/token`** per outbound table), **resolves user/session**, **persists OAuth tokens** to **MariaDB** (`oauth_*` models per main-backend profile).
7. Browser is **returned to the SPA** at **`/login/authorize`** (**main-frontend** route per profile) to complete client-side session establishment; optional **`redirectSuccessfulAuthentication` / `redirectFailedAuthentication`** for **token query** patterns (`Redirect.ts`).
8. Subsequent API calls use **`Authorization: Bearer`** (from cookies per **`HttpRequest.ts`**). **401** responses trigger **user clear** and **re-login** path.

### B) **Subscription** portal user (direct SSO web login)

1. User hits **`GET /sso/login`** on **orangekloud-subscription** → **`AuthController`** reads **`settings.sso`** (DB JSON: `sso_server_url`, `ep_authorize`, token endpoint paths, redirect URLs—**not** in `.env` per subscription profile).
2. Browser redirects to **SSO** OAuth **authorize** URL with **`client_id`**, **`redirect_url`**, **`state`** (subscription profile “SSO web login” flow).
3. After user authenticates at **orangekloud-sso**, browser returns to **subscription** **`GET /callback`** with **`code`**.
4. **Subscription** exchanges **`code`** via **`POST`** (form) to **`{sso_server_url}{ep_oauth_tokens}`** (`AuthController`), stores tokens in **session**.
5. **`SsoTokenToSession`** may restore or align session from **`_em_id`** cookie for parity with other apps.

### C) **SSO-first** browser login → **main-frontend** with token query (documented on SSO)

1. User authenticates via **orangekloud-sso** **web** guards (classic or Google) per SSO profile.
2. **`LoginController`** creates a Passport **personal access** token and sets **cookie** semantics for **`_em_id`** (or `AUTH_TOKEN_COOKIE_NAME`).
3. If target is **main-frontend**-related, SSO may redirect to **`{MAIN_FRONTEND_URL}/login/authorize?token=&state=`** so the SPA adopts the token.

### D) **Optional multi-hop bootstrap** (`oauth_chain`)

1. When **`OAUTH_CLIENT_BOOTSTRAP_ENABLED`** and related env/config are set on **orangekloud-sso**, **`OAuthClientBootstrapChain`** may send the browser through **ordered** URLs: **`MAIN_BACKEND_OAUTH_LOGIN_URL`**, **subscription** **`{SUBSCRIPTION_URL}{SUBSCRIPTION_SSO_LOGIN_URL}`**, **`SERVICE_MANAGER_LOGIN_URL`**, using signed **`oauth_chain=`** (`orangekloud-sso` profile).
2. Consumers must validate **HMAC** with shared **`OAUTH_CLIENT_BOOTSTRAP_SECRET`** (`orangekloud-sso` profile).

### E) **Admin** (documented limitation)

1. **orangekloud-sso** profile: **Administrators authenticate in orangekloud-subscription**; SSO **`/admin/login`** routes are **not** the production admin entry.
2. **INFERRED:** Admin login path for platform administration is **primarily subscription web/admin session**, not the main-frontend `/login` SPA flow—**NEEDS CONFIRMATION** for your org’s exact URLs and roles.

## Endpoints

| From | To | Method | Endpoint | Purpose | Auth / Headers | Confidence |
|---|---|---|---|---|---|---|
| Browser | `emobiq-main-frontend` | Navigate | `/login` | Start login | SPA session/cookies | High |
| Browser | `emobiq-main-frontend` | Navigate | `/login/authorize` | OAuth callback / token handoff route | Per SPA | High |
| Browser | `emobiq-main-backend` | `GET` | `/v1/oauth/login` | Redirect to IdP to start OAuth | Profile: selected routes unauthenticated | High |
| Browser | `emobiq-main-backend` | `GET` | `/v1/callback` | OAuth redirect **callback** with `code` | Profile: No OAuth middleware on callback | High |
| `emobiq-main-backend` | SSO server | **Various** | Paths from `api.sso` (e.g. token exchange) | Code → tokens, user resolution | Config-driven | High |
| Browser | `orangekloud-sso` | `GET` | `/oauth/authorize` | Passport authorization | `web` session (Passport standard) | High |
| Browser | `orangekloud-sso` | `POST` | `/oauth/token` | Passport token issuance | OAuth2 client / grant | High |
| Browser | `orangekloud-sso` | `GET` / `POST` | `/login`, `/login/google`, `/login/google/callback`, etc. | Web/session login | Guest/session per route | High |
| Browser | `orangekloud-sso` | **Navigate** | `{MAIN_FRONTEND_URL}/login/authorize` with `token`, `state` query | Pass token into main-frontend SPA | Query params documented on SSO profile; exact SPA handling **NEEDS CONFIRMATION** per deploy | Medium |
| Trusted HTTP client | `orangekloud-sso` | `POST` | `/api/sso/login` | Credential check + client validation (`client_id` / `client_secret` in body) | **Not** Bearer | High |
| Browser | `orangekloud-subscription` | `GET` | `/sso/login` | Redirect to SSO authorize | None | High |
| Browser | `orangekloud-subscription` | `GET` | `/callback` | Subscription OAuth callback (code) | Session | High |
| `orangekloud-subscription` | SSO server | `POST` | `{sso_server_url}{ep_oauth_tokens}` (from `settings.sso`) | Exchange authorization code | Form POST per `AuthController` | High |
| `orangekloud-subscription` | SSO server | `GET` | `{sso_server_url}{ep_user}` (**path name inferred** from subscription profile) | Fetch user with Bearer | Bearer token | Medium |
| API consumer | `emobiq-main-backend` | `POST` | `/v1/oauth/logout` | Revoke / end OAuth session server-side | Typically Bearer (`OAuthTokenFull` group per profile—not exhaustively enumerated here) | High |
| API client | `emobiq-main-backend` | `POST` | `/v1/oauth/tokens/introspect` | Validate access token | Bearer (per consumer) | High |
| Browser | `emobiq-main-frontend` | Navigate | **SSO URL** (from `REACT_APP_SSO_URL`) | Logout / `_em_id` clearance | Per `Logout.tsx` | High |
| Browser | `emobiq-service-manager` | Navigate | **`/login`** | Start web login (redirect toward SSO OAuth) | Unauthenticated redirect | High |
| Browser | `emobiq-service-manager` | `GET` | **`/v1/callback-service-manager`** | OAuth authorization-code callback for service-manager web | Code in query; exchanges via SSO per profile | High |
| `orangekloud-sso` | `orangekloud-subscription` | `POST` | `{SUBSCRIPTION_URL}/api/sso/complete-signup` | Post-signup onboarding | `X-SSO-Signup-Api-Key` | High |
| `orangekloud-sso` | `orangekloud-subscription` | `POST` | `{SUBSCRIPTION_URL}/api/sso/email-verification-status` | Email verification reconciliation | **NEEDS CONFIRMATION** exact body | Medium |

## Data / State

| Repo | Table / Model / Redis / Queue / Storage / File | Purpose | Read / Write | Confidence |
|---|---|---|---|---|
| `emobiq-main-backend` | MariaDB `users`, `companies`, and related platform tables | User/company records for authenticated platform API | Read / Write on login sync flows | High |
| `emobiq-main-backend` | MariaDB **`oauth_*`** (Passport-related persistence named in profile: e.g. oauth token models under `golang/app/model/mariadb/`) | Persist OAuth tokens after callback | Write on successful OAuth | High |
| `orangekloud-sso` | `users` | Credentials, verification, IdP linkage | Read / Write | High |
| `orangekloud-sso` | `oauth_clients`, `oauth_access_tokens`, `oauth_refresh_tokens`, `oauth_auth_codes`, `oauth_personal_access_clients` | Laravel Passport | Read / Write | High |
| `orangekloud-subscription` | `users` (includes `idp_id`) | Subscription-side user identity | Read / Write | High |
| `orangekloud-subscription` | `settings` column **`sso`** (JSON) | Authorize/token URLs, client IDs, `ep_*` paths | Read (runtime), Write (admin UI) | High |
| `orangekloud-subscription` | Laravel **session** store | Web login session after `/callback` | Read / Write | High |
| `emobiq-main-frontend` | Browser **cookies** (Bearer / API access) | API auth after login | Read / Write | High |
| — | **`_em_id`** cookie (default name per SSO `AUTH_TOKEN_COOKIE_NAME`) | Cross-app token sharing on SSO domain | Read / Write | High |
| `emobiq-service-manager` | PostgreSQL **`user`** (and related ESM entities per profile) | Local ESM account after SSO/plugin flows | Read / Write when user is created or synced | Medium |

## Async, queues, Redis, and cron (mostly orthogonal)

| Repo | Mechanism | Relevance to login | Confidence |
|---|---|---|---|
| `emobiq-main-backend` | Optional **Redis** (`database.json` → connector rate limit) | **Not** on the browser OAuth login path | High |
| `orangekloud-subscription` | Scheduler (`subscriptions:status`, `subscriptions:alert`) and **queue** mail jobs | **Not** required to complete OAuth web login | High |
| `orangekloud-sso` | Queue driver default `sync` in sample; no app cron in profile | **Not** on synchronous login | High |
| `emobiq-service-manager` | PM2 / Node process (no separate login queue in profile) | Session + OAuth callbacks are **synchronous HTTP** | High |

## Config

| Repo | Config / Env Var | Purpose | Confidence |
|---|---|---|---|
| `emobiq-main-frontend` | `REACT_APP_MAIN_BACKEND_URL` | Main API + OAuth base for login URL | High |
| `emobiq-main-frontend` | `REACT_APP_MAIN_BACKEND_VERSION`, `REACT_APP_MAIN_BACKEND_LOGIN_PATH` | Segments for OAuth login path (`Redirect.ts`) | High |
| `emobiq-main-frontend` | `REACT_APP_SSO_URL` | Logout redirect | High |
| `emobiq-main-backend` | `golang/config/api.json` → `sso`, `main`, `sso.redirect_url` (per **Things Other Repos Depend On**) | IdP URLs, callback registration, frontend redirects | High |
| `emobiq-main-backend` | `golang/config/application.json` (token lifetimes, `sessionSecret`, etc.) | Session/JWT-related timings | Medium |
| `orangekloud-subscription` | `MAIN_BACKEND_URL`, `MAIN_BACKEND_API_KEY` | Sync/API key flows (adjacent to login, not every user login) | High |
| `orangekloud-subscription` | `MAIN_FRONTEND_URL` | Post-login redirects | High |
| `orangekloud-subscription` | `OAUTH_CLIENT_BOOTSTRAP_SECRET` | Validates `oauth_chain` on SSO login | High |
| `orangekloud-subscription` | DB `settings.sso` JSON | SSO server URL, OAuth path keys (`ep_*`), client secrets | High |
| `orangekloud-sso` | `MAIN_FRONTEND_URL`, `CLIENT_URL` | Post-login targets | High |
| `orangekloud-sso` | `SUBSCRIPTION_URL`, `SUBSCRIPTION_SIGNUP_API_KEY` | Subscription API calls after signup / verification | High |
| `orangekloud-sso` | `OAUTH_CLIENT_BOOTSTRAP_ENABLED`, `OAUTH_CLIENT_BOOTSTRAP_SECRET` | Multi-hop redirect chain | High |
| `orangekloud-sso` | `MAIN_BACKEND_OAUTH_LOGIN_URL`, `SUBSCRIPTION_SSO_LOGIN_URL`, `SERVICE_MANAGER_LOGIN_URL` | Bootstrap chain targets | High |
| `orangekloud-sso` | `PASSPORT_PERSONAL_ACCESS_CLIENT_ID`, `PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET` | Personal-access token for `_em_id` | High |
| `orangekloud-sso` | `AUTH_TOKEN_COOKIE_NAME`, `AUTH_TOKEN_COOKIE_TTL_MINUTES` | Cookie naming/TTL | High |
| `orangekloud-sso` | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI` | Google IdP federation | High |
| `emobiq-service-manager` | `config/config.js` (`platform.ssoUrl`, `platform.mainBackendUrl`, etc.; sample `config/config.js.sample`) | SSO OAuth + token introspection via main-backend | High |

## Failure Cases

| Case | Expected Behavior | Where Handled | Confidence |
|---|---|---|---|
| Unauthenticated access to protected main-frontend route | Redirect to **`/login`** | `ProtectedRoute` / `Route.tsx` | High |
| **401** from main-backend on authenticated axios instance | Clear user state; navigate toward login | `HttpRequest.ts` | High |
| IdP / network failure during **`/v1/oauth/login`** or callback | User sees error or incomplete redirect; **NEEDS CONFIRMATION** UX | Browser + main-backend `oauth` | Medium |
| **Callback URL mismatch** | OAuth error from IdP or failed code exchange | IdP registration vs `api.sso.redirect_url` | High |
| Email not verified (SSO web user) | Redirect to subscription **waiting-for-verification** or reconcile via **`email-verification-status`** | `SubscriptionEmailVerificationRedirect.php` | High |
| Invalid **`oauth_chain`** signature | Bootstrap hop should fail validation | **`OAuthClientBootstrapChain`** consumers | Medium |
| Subscription **`settings.sso`** misconfigured | `/sso/login` redirects broken or token exchange fails | `AuthController`, admin settings | High |

## Debugging Notes

- **Main-frontend:** Confirm **`getOAuthLoginUrl()`** output matches **main-backend**’s registered **`GET /v1/oauth/login`** and that **`/login/authorize`** is the callback route configured in **`Route.tsx`**.
- **Main-backend:** Verify **`golang/config/api.json`** **`sso.redirect_url`** matches IdP app’s allowed redirect (**`.../v1/callback`** per profile).
- **Subscription:** Inspect **`settings`** table **`sso`** JSON for **`sso_server_url`** and **`ep_*`** paths; compare with SSO **`Passport`** client redirect URIs.
- **Cookie alignment:** **`_em_id`** / **`AUTH_TOKEN_COOKIE_NAME`** and domain flags must match across **SSO**, **subscription** (`SsoTokenToSession`), and **main-frontend** expectations (profile comments).
- **Admin:** If debugging **admin** access, start from **orangekloud-subscription** admin sign-in, not SSO **`/admin/login`**, per SSO profile.
- **Service-manager web login:** Trace **`GET /login` → SSO authorize → `GET /v1/callback-service-manager`** and confirm **`platform.*`** URLs in `config/config.js` match **Passport** client redirect URIs (separate from main-backend **`/v1/callback`**).
- **`oauth_chain`:** When enabled, confirm each hop’s URL and shared **`OAUTH_CLIENT_BOOTSTRAP_SECRET`** across **SSO**, **subscription** (`AuthController`), and any consumer on **service-manager** (per respective profiles).

## Unknowns / Needs Confirmation

- Exact **admin** login URLs and role gates when **not** using main-frontend **`/login`** (subscription vs SSO Blade) for each deployment.
- Full **`settings.sso`** JSON key set per environment (subscription profile §12).
- Whether legacy **`api_token`** / **`auth:api` token driver** still applies to any **subscription** APIs used during login-adjacent flows.
- Intended auth for **`POST /verify/token/user`** (web) vs **`POST /api/sso/verify/token/user`** on **orangekloud-sso** (duplicate shapes).
- Production status of **`GET /callback`** test route on **orangekloud-sso** (hard-coded local token exchange).

---

## Suggested follow-up updates (maps)

- **`repo-hub/maps/service-map.md`:** The **Login (OAuth / SSO)** row already points here. When next editing the map, consider adding **`emobiq-service-manager`** to that flow row (bootstrap chain + web **`/login`** / **`/v1/callback-service-manager`**) if you want the table to match this doc’s full repo set.
- **`repo-hub/maps/database-map.md`:** Add or extend a **Login / Identity** cross-reference for **`oauth_*`** (main-backend MariaDB), **Passport tables** (SSO), **`users.idp_id`** / **`settings.sso`** (subscription), and **ESM `user`** (PostgreSQL, service-manager) if not already grouped that way.
