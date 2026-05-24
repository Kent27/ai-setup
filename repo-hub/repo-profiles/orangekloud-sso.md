# Repo Profile: orangekloud-sso

Last updated: 2026-05-07
Confidence: High

## 1. Purpose

Laravel SSO / identity Provider (IdP) for Orangekloud: user signup and login (email/password and Google OAuth), Laravel Passport OAuth2 (authorization + tokens), session-based redirects to main frontend and subscription flows, cookie-based sharing of access tokens (`_em_id`), and optional multi-app OAuth “bootstrap chain” across main-backend, subscription, and service-manager. Includes an authenticated admin Blade UI for OAuth client and user oversight; in current operations admins sign in through the sibling **`orangekloud-subscription`** app (paired with this repo), **not** through local `/admin/login` routes here (those are commented out).

## 2. Tech Stack

- **Language:** PHP (Composer requires `^7.3|^8.0`; README targets PHP 8.2).
- **Framework:** Laravel 8 (`laravel/framework` ^8.75).
- **Runtime:** PHP-FPM / Apache-style deployment (README documents Apache + MariaDB on macOS Homebrew).
- **Package managers:** Composer (PHP), npm (frontend assets via Laravel Mix).
- **Database:** MySQL/MariaDB (`DB_CONNECTION=mysql` in `.env.example`).
- **Other important dependencies:** Laravel Passport (OAuth2 API tokens), Laravel UI (legacy auth scaffolding), Guzzle/`Illuminate\Support\Facades\Http` for outbound HTTP, Vue 2 + Laravel Mix + Bootstrap 5 on the frontend.

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `routes/web.php` | Web/session routes: login, Google OAuth, SSO logout, profile, admin Blade routes, embedded test `/callback`. |
| `routes/api.php` | JSON SSO API prefixed with `/api` (`ClientsController`). |
| `app/Http/Controllers/Clients/ClientsController.php` | Core SSO REST: login, register, user CRUD-style ops, logout, subscription mirror. |
| `app/Http/Controllers/Auth/LoginController.php` | Google SSO, remember-me, full logout, post-login redirects, `_em_id` cookie, OAuth bootstrap redirect. |
| `app/Support/SubscriptionEmailVerificationRedirect.php` | Unverified-email handling + subscription reconciliation HTTP call. |
| `app/Support/OAuthClientBootstrapChain.php` | HMAC-signed `oauth_chain=` token for sequential browser redirects to sibling apps. |
| `app/Passport/PassportClient.php` | Custom Passport client model (`Passport::useClientModel` in `AuthServiceProvider`). |
| `app/Providers/AuthServiceProvider.php` | Registers Passport routes, token TTLs, Passport client model. |
| `config/app.php` | Cross-app URLs and bootstrap flags (`main_frontend_url`, `subscription_url`, `oauth_client_bootstrap_*`, etc.). |
| `config/auth.php` | `web`, `admin` (session), `api` (Passport); token/remember cookie names and TTL env hooks. |
| `resources/views/vendor/passport/authorize.blade.php` | Overridden Passport authorization UI. |

## 4. Exposed Endpoints

All API SSO routes live under **`/api`** (`RouteServiceProvider` prefix).

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/oauth/authorize` | OAuth2 authorization endpoint (Passport). | Laravel Passport (`AuthServiceProvider` → `Passport::routes()`) | Browser/session (`web` middleware on Passport authorize flow — **standard Passport behavior**) |
| `POST` | `/oauth/token` | OAuth2 token issuance. | Laravel Passport | Client credentials / user credentials per OAuth2 |
| Passport management routes | `/oauth/*` (scopes, clients UI API as registered) | Passport-registered tooling | Laravel Passport | Varies |
| `GET` | `/api/user` | Current Bearer user payload. | `routes/api.php` closure | Yes (`auth:api`) |
| `GET` | `/api/sso/user` | SSO-shaped user JSON. | `ClientsController::getUser` | Yes (`auth:api`) |
| `POST` | `/api/sso/user/subscription` | Mirror subscription/licensing fields onto user. | `ClientsController::updateSubscription` | Yes (`auth:api`) |
| `POST` | `/api/sso/login` | Credential check + OAuth client validation (delegates OAuth token issuance to `/oauth/token` in practice — see controller comments). | `ClientsController::login` | No (`client_id` / `client_secret` in body — **NOT** Bearer) |
| `POST` | `/api/sso/register` | Register user for trusted OAuth client. | `ClientsController::register` | No (`client_id` / `client_secret`) |
| `POST` | `/api/sso/user/reset_password` | Reset password for trusted OAuth client. | `ClientsController::resetPassword` | No (`client_id` / `client_secret`) |
| `POST` | `/api/sso/user/delete` | Delete user by `user_id` for trusted OAuth client. | `ClientsController::deleteUser` | No (`client_id` / `client_secret`) |
| `POST` | `/api/sso/user/update` | Trusted client updates user (e.g. `mark_email_verified`). | `ClientsController::updateUser` | No (`client_id` / `client_secret`) |
| `POST` | `/api/sso/open/idp` | Resolve IdP-backed user id for external/migrated username. | `ClientsController::getIDPByUsername` | No (`client_id` / `client_secret`) |
| `POST` | `/api/sso/verify/token/user` | Health-style check returning authenticated user id. | `routes/api.php` closure | Yes (`auth:api`) |
| `POST` | `/api/sso/logout` | Revoke Passport tokens for current user. | `ClientsController::logout` | Yes (`auth:api`) |

**Web/session (non–`/api` prefix, `web` middleware unless noted)**

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/` | Landing/welcome | `WelcomeController` | Unknown / public |
| `GET`/`POST` | `/login`, `/register`, `/password/*`, etc. | Laravel `Auth::routes()` defaults | Laravel UI / auth controllers | Guest/session per route |
| `GET` | `/login/google` | Start Google OAuth | `LoginController::redirectToGoogle` | Typically guest |
| `GET` | `/login/google/callback` | Google OAuth callback | `LoginController::handleGoogleCallback` | Public callback |
| `GET` | `/login/remember` | Auto-login via encrypted remember cookie | `LoginController::loginWithRemember` | Guest-oriented |
| `GET` | `/sso/logout/full` | Cookie + SSO session teardown for client apps | `LoginController::fullLogout` | Varies |
| `POST` | `/verify/token/user` | Token/user presence check (`ClientsController`) | `ClientsController::verifyTokenUser` | Unknown / **likely session or open — confirm deployment** |
| `GET` | `/callback` | **Hard-coded test** authorization_code → token exchange (`Http::post` local) | `routes/web.php` closure | **Local test artifact** |

**Admin (`auth:admin` middleware groups)**

| Method | Path (examples) | Purpose | Main handler | Auth |
|---|---|---|---|---|
| `GET` | `/admin/dashboard`, `/admin/clients`, `/admin/users`, `/admin/security`, … | Admin dashboard and management | Various `Admin\\*` controllers | Yes (`auth:admin`) |

**Admin authentication (current ops):** Admins authenticate in **`orangekloud-subscription`**, which is designed to operate together with this SSO repo. **`AdminLoginController` routes (`/admin/login`) are commented out** in `routes/web.php`; that path is **not used** as the production entry today. **`AdminMiddleware`** still redirects failed `admin` guard checks to `admin/login`, which reflects legacy wiring alongside the inactive local login routes—not a contradiction with subscription-based admin sign-in once a valid `auth:admin` session exists.

## 5. Outbound API Calls

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| Google OAuth | `POST` | `https://oauth2.googleapis.com/token` | Exchange authorization code | `LoginController.php` |
| Google | `GET` | `https://www.googleapis.com/oauth2/v2/userinfo` | User profile during Google login | `LoginController.php` |
| Google | Browser redirect | `https://accounts.google.com/o/oauth2/v2/auth` | Start Google SSO | `LoginController.php` |
| Subscription app | `POST` | `{SUBSCRIPTION_URL}/api/sso/complete-signup` | Post-signup / Google-signup onboarding (`X-SSO-Signup-Api-Key`) | `LoginController.php`, `RegisterController.php` |
| Subscription app | `POST` | `{SUBSCRIPTION_URL}/api/sso/email-verification-status` | Reconcile email verification by `idp_id` | `SubscriptionEmailVerificationRedirect.php` |

**Redirects (browser hops, configured URLs — not outbound HTTP from server in all cases)**

- After login, SSO may redirect the browser through `MAIN_BACKEND_OAUTH_LOGIN_URL`, then subscription `{SUBSCRIPTION_URL}{SUBSCRIPTION_SSO_LOGIN_URL}`, then `SERVICE_MANAGER_LOGIN_URL`, using signed `oauth_chain=` when bootstrap is enabled — `LoginController.php`, `OAuthClientBootstrapChain.php`, `config/app.php`.
- Unverified SSO users → `{SUBSCRIPTION_URL}/account/waiting-for-verification` — `SubscriptionEmailVerificationRedirect.php`.

**Embedded test traffic**

| Target | Method | Endpoint | Purpose | Source |
|---|---|---|---|---|
| Local SSO | `POST` | `http://127.0.0.1:8000/oauth/token` | Hard-coded test token exchange incl. client_secret | `routes/web.php` `/callback` |

## 6. Database / Models / Tables

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| `users` | End-user + **admin guard** shares this table (`App\Models\User` vs `App\Models\Admin\User`): credentials, subscription JSON-like field, migration metadata, verification | Read / Write | `app/Models/User.php`, `app/Models/Admin/User.php`, `database/migrations/2014_10_12_000000_create_users_table.php` |
| `oauth_clients`, `oauth_access_tokens`, `oauth_refresh_tokens`, `oauth_auth_codes`, `oauth_personal_access_clients` | Laravel Passport persistence | Read / Write | `database/migrations/2016_06_01_*.php`, `ClientsController.php`, Passport |
| `password_resets` | Password reset tokens | Read / Write | `database/migrations/2014_10_12_100000_create_password_resets_table.php` |
| `personal_access_tokens` | Sanctum-style table shipped with Laravel | Unknown usage | Migration present |
| `failed_jobs` | Failed queue jobs schema | Write when jobs fail | `database/migrations/2019_08_19_000000_create_failed_jobs_table.php` |
| `settings` | App metadata incl. optional `eps_url` | Read / Write | `SettingsModel`, `database/migrations/2024_12_02_085438_create_settings_table.php` |

## 7. Jobs / Queues / Cron / Workers

No scheduled tasks are defined in `app/Console/Kernel.php` (`schedule` is empty). Default queue driver in `.env.example` is `sync`. No `app/Jobs` classes were found.

No jobs, queues, cron tasks, or workers were found.

## 8. Configuration & Environment

Describe where integration/runtime settings live in this repo.

### Primary: `.env` + Laravel `config/*`

Brief: Laravel-standard pattern — values from `.env` are read via `env()` into `config/app.php`, `config/auth.php`, `config/services.php`. **Never commit real `.env`;** `.env.example` lists integration-related keys without secrets.

| Key / Name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `APP_URL`, `APP_KEY` | App base URL / encryption | `.env.example`, `config/app.php` |
| `CLIENT_URL`, `MAIN_FRONTEND_URL` | Default / main frontend post-login redirects | `.env.example`, `config/app.php`, `LoginController.php` |
| `SUBSCRIPTION_URL`, `SUBSCRIPTION_SIGNUP_API_KEY` | Subscription base URL + header for SSO APIs (`X-SSO-Signup-Api-Key`) | `.env.example`, `config/app.php`, `LoginController.php`, `RegisterController.php`, `SubscriptionEmailVerificationRedirect.php` |
| `OAUTH_CLIENT_BOOTSTRAP_ENABLED`, `OAUTH_CLIENT_BOOTSTRAP_SECRET` | Signed multi-hop OAuth chain toggle + HMAC secret | `.env.example`, `config/app.php`, `LoginController.php` |
| `MAIN_BACKEND_OAUTH_LOGIN_URL`, `SUBSCRIPTION_SSO_LOGIN_URL`, `SERVICE_MANAGER_LOGIN_URL` | Ordered bootstrap redirect targets | `.env.example`, `config/app.php`, `LoginController.php` |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `GOOGLE_REDIRECT_URI` | Google OAuth app | `.env.example`, `config/services.php`, `LoginController.php` |
| `PASSPORT_PERSONAL_ACCESS_CLIENT_ID`, `PASSPORT_PERSONAL_ACCESS_CLIENT_SECRET` | Passport personal-access client for `_em_id` minting | `.env.example`, `LoginController.php` |
| DB `DB_*` | MySQL connection | `.env.example`, Laravel default |

### Other / Secondary

| Variable / Key | Purpose | Used In |
|---|---|---|
| `AUTH_TOKEN_COOKIE_NAME`, `AUTH_TOKEN_COOKIE_TTL_MINUTES` | Bearer cookie name/time for SSO domain | `config/auth.php`, `LoginController.php` |
| `AUTH_REMEMBER_COOKIE_NAME`, `AUTH_REMEMBER_COOKIE_TTL_DAYS` | Remember-me encrypted cookie | `config/auth.php`, `LoginController.php` |

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| MySQL/MariaDB | Database | Users, Passport tables, settings | `.env.example`, migrations |
| Google OAuth | Auth / IdP federation | Email-based Google signup/login | `LoginController.php`, `config/services.php` |
| Subscription app | HTTP API | `complete-signup`, email-verification mirror, browser waiting pages | `.env.example`, `LoginController.php`, `RegisterController.php`, `SubscriptionEmailVerificationRedirect.php` |
| Main frontend | Redirect target | `/login/authorize`, `/ai`, return paths | `config/app.php`, `LoginController.php` |
| Main backend | Redirect / OAuth bootstrap | Optional first hop OAuth URL | `.env.example`, `LoginController.php` |
| Service manager | Redirect / OAuth bootstrap | Optional hop in bootstrap chain | `.env.example`, `LoginController.php` |
| Laravel Passport | OAuth2 | Access tokens for API SSO + clients | `composer.json`, `AuthServiceProvider`, `oauth_*` migrations |
| **orangekloud-subscription** (sibling repo) | Web app / pairing | Administrative user sign-in and flows that integrate with SSO; admins do not rely on SSO’s `/admin/login` | Operational note (confirmed); `routes/web.php` admin login commented |

## 10. Main Flows

### Flow: Email/password SSO API (trusted client)

1. Client validates with `client_id` + `client_secret` against `oauth_clients` (`ClientsController`).
2. User may register (`POST /api/sso/register`), reset password (`/api/sso/user/reset_password`), or log in (`/api/sso/login` verifies credentials via `Auth`).
3. Client obtains OAuth tokens through standard Passport `POST /oauth/token` (**per controller documentation** referencing password grant-style usage).
4. Protected calls use Bearer token (`auth:api`) for `/api/sso/user`, subscription updates, logout, etc.

### Flow: Browser login → main frontend with token cookie

1. User authenticates via `web` guards (classic login or Google).
2. `LoginController` creates a Passport **personal access** token and sets HTTP-only-visible cookie semantics per `attachTokenCookie` (non–httpOnly `_em_id` for JS readability per controller options).
3. If target URL is recognized as main-frontend-related, SSO resolves `resolvePostLoginTargetUrl()` and may redirect to `{MAIN_FRONTEND}/login/authorize?token=&state=` so the SPA can adopt the token (`LoginController.php`).
4. Optional: `oauth_chain` bootstrap walks main-backend → subscription SSO path → service-manager before final URL (`OAuthClientBootstrapChain`).

### Flow: Email verification parity with Subscription

1. SSO checks `users.email_verified_at`.
2. If empty, SSO may POST to subscription `{SUBSCRIPTION_URL}/api/sso/email-verification-status` with `idp_id` — on `verified=true`, SSO sets `email_verified_at` (`SubscriptionEmailVerificationRedirect.php`).
3. Otherwise user is redirected browser-side to subscription `account/waiting-for-verification` with optional `email` query parameter.

### Flow: OAuth2 authorization code (third-party / first-party apps)

1. Client redirects browser to SSO `/oauth/authorize` with client id/redirect_uri (standard Passport).
2. User authorizes (`resources/views/vendor/passport/authorize.blade.php`).
3. Client exchanges code at `/oauth/token`.

### Flow: Admin Blade UI (`/admin/*`)

1. Administrators sign in through **`orangekloud-subscription`**, which works with this SSO deployment.
2. With an **`auth:admin`** session satisfied, SSO’s guarded `/admin/dashboard`, OAuth client management, and related Blade routes operate as coded in `routes/web.php` (`auth:admin` groups).
3. SSO’s **`/admin/login`** route group is intentionally disabled in routing today; **`AdminMiddleware`**’s redirect target `admin/login` is leftover from older local-login expectations.

## 11. Things Other Repos Depend On

- **Stable SSO JSON API paths** under `/api/sso/*` and Bearer-protected mirrors (`ClientsController`).
- **`POST /oauth/token`** Passport token semantics and Passport table layout (custom client model `app/Passport/PassportClient.php`).
- **Cookie `_em_id` (default)** or `AUTH_TOKEN_COOKIE_NAME` alignment with sibling apps consuming the SSO domain cookie.
- **Browser redirect URLs:** main-frontend **`/login/authorize`** handshake with `token` + `state` query parameters (`LoginController.php`).
- **Subscription contract:** `{SUBSCRIPTION_URL}/api/sso/complete-signup` request/response expectations and `X-SSO-Signup-Api-Key` header naming; **`/api/sso/email-verification-status`** payload `{ idp_id }` → `{ verified }`.
- **OAuth bootstrap:** consuming apps must honor **`oauth_chain`** query forwarding and validate HMAC (`OAuthClientBootstrapChain.php`) — same `OAUTH_CLIENT_BOOTSTRAP_SECRET` across consumers.
- **User id alignment:** SSO user `id` is treated as **`idp_id`** when calling subscription.

## 12. Unknowns / Needs Confirmation

- **NEEDS CONFIRMATION:** Intended auth model for **`POST /verify/token/user`** on `web.php` versus **`POST /api/sso/verify/token/user`** (duplicate behavior shapes with different stacks).
- **NEEDS CONFIRMATION:** Whether `AutoAuthorizeController` is missing or relocated — it is imported in `AuthServiceProvider.php` but no matching controller file appeared in scan (may be dead import).
- **INFERRED:** Production removal or firewall of test route **`GET /callback`** carrying hard-coded client credentials (`routes/web.php`).
