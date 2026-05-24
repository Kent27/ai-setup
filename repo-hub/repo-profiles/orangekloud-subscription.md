# Repo Profile: orangekloud-subscription

Last updated: 2026-05-07
Confidence: Medium–High

## 1. Purpose

Laravel application for **Orangekloud subscription management**: client and admin UIs for plans, add-ons, checkout, Stripe payments, credits and holds, companies and users, SSO-based login, and JSON APIs for other services (SSO signup completion, subscription/RtL lookups, credits). It syncs selected user/company/API-key data to a **main backend** and can call **service-manager** and the **SSO** server using configured URLs.

## 2. Tech Stack

- Language: PHP (^8.1 per `composer.json`; PHP 8.2 is typical for local Homebrew installs)
- Framework: Laravel ^10.10
- Runtime: PHP-FPM / `php artisan serve` (typical); Apache noted in README
- Package manager: Composer (PHP), npm (Vite assets)
- Database: MySQL / MariaDB (`DB_CONNECTION=mysql` in `.env.example`)
- Other important dependencies: Guzzle (HTTP client), Spatie Laravel Permission, Cartalyst Stripe Laravel, Yajra DataTables, Maatwebsite Excel, Laravel Sanctum (in `composer.json`; `GET /api/user` uses `auth:api` with legacy **token** guard per `config/auth.php`), Laravel UI, Pragmarx Google2FA

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `routes/api.php` | Prefix `/api` — machine-oriented JSON routes (SSO signup, subscription, credits) |
| `routes/web.php` | Browser sessions — SSO, client portal, admin, cart, payments |
| `app/Http/Controllers/Api/` | `SubscriptionController`, `CreditsController`, `SsoSignupController` |
| `app/Http/Controllers/SSO/AuthController.php` | OAuth authorization-code SSO login, callback, session, editor launch |
| `app/Services/MainBackendSyncService.php` | Outbound calls to main backend (`/sso/company|user/...`, API keys) |
| `app/Services/EsmUserService.php` | Outbound `POST` to service-manager `esm-user-create` |
| `app/Services/SsoSignupCompletionService.php` | SSO signup completion; optional `POST` to SSO `api/sso/user/update` |
| `app/Helpers/AppHelper.php` | Helpers including `getTokenStatus` (SSO token verify via DB-driven endpoints) |
| `app/Http/Middleware/SsoTokenToSession.php` | Restores session from cookie (see comment: `_em_id`, alignment with main-frontend/service-manager) |
| `config/services.php` | `MAIN_BACKEND_*`, `SERVICE_MANAGER_*`, `SSO_SIGNUP_API_KEY`, mail/third-party placeholders |
| `config/app.php` | `main_frontend_url` / `oauth_client_bootstrap_secret` (env-backed) |
| `database/migrations/` | Schema for users, subscriptions, orders, credits, settings, etc. |
| `app/Console/Kernel.php` | Scheduler: subscription status + alert commands |
| `app/Jobs/` | Mail jobs (`VerifyEmailJob`, `WelcomeMailJob`, subscription/cancel/alert mail jobs) |
| `.env.example` | Canonical env template for this repo |
| `vite.config.js`, `resources/` | Frontend assets (Vite, Bootstrap) |

## 4. Exposed Endpoints

All `routes/api.php` routes use the `api` middleware group and URI prefix **`/api`**.

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/api/user` | Authenticated user (`api` guard: token driver in `config/auth.php`) | `routes/api.php` | Yes (`auth:api`) |
| `POST` | `/api/sso/complete-signup` | Finish signup in subscription DB after SSO created the user | `app/Http/Controllers/Api/SsoSignupController.php` | Yes (`sso.signup.api`: `X-SSO-Signup-Api-Key` or `api_key`; matches `SSO_SIGNUP_API_KEY`) |
| `POST` | `/api/sso/email-verification-status` | Email verification status for SSO flow | `SsoSignupController` | Yes (`sso.signup.api`) |
| `POST` | `/api/user/subscription` | Subscription details for caller | `app/Http/Controllers/Api/SubscriptionController.php` | Yes (`api_token`: Bearer token validated against SSO) |
| `POST` | `/api/open/rtl` | RtL details by license key | `SubscriptionController` | No |
| `POST` | `/api/open/commercial` | Commercial key / subscription date | `SubscriptionController` | No |
| `POST` | `/api/open/get_subscription_attr_by_idp` | Subscription attributes by IdP id | `SubscriptionController` | No |
| `POST` | `/api/credits/hold` | Create credit hold | `app/Http/Controllers/Api/CreditsController.php` | Yes (`token.or.api.key` + `whitelist.ip`) |
| `POST` | `/api/credits/release-hold` | Release hold | `CreditsController` | Yes (`token.or.api.key` + `whitelist.ip`) |
| `POST` | `/api/credits/charge` | Charge credits | `CreditsController` | Yes (`token.or.api.key` + `whitelist.ip`) |
| `GET` | `/api/credits/balance` | Credit balance | `CreditsController` | Yes (`token.or.api.key`; no IP whitelist) |

**Web (`routes/web.php`):** large Blade application (admin, client, cart, payments). Cross-service highlights:

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/sso/login` | Redirect to SSO authorize URL | `app/Http/Controllers/SSO/AuthController.php` | No |
| `GET` | `/callback` | OAuth callback (code → token) | `AuthController` | Session |
| `GET` | `/sso/user` | Loads SSO user into session | `AuthController` | Session |
| `GET` | `/sso/session/clear` | Clears subscription session / cookie (SSO logout chain) | `AuthController` | No |
| `GET` | `/sso/token/validate` | Token validation helper | `app/Http/Controllers/OAuthValidateController.php` | Unknown / query-based |
| `GET` / `POST` | `/api/company/profile` | Company profile JSON for onboarding | `app/Http/Controllers/Client/Account/OrganizationController.php` | Yes (`auth` — web session) |

Additional authenticated routes (client `auth`, admin `auth:admin`, etc.) are defined in `routes/web.php` for dashboards, CRUD, settings, imports — not enumerated here.

## 5. Outbound API Calls

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| Main backend (`config('services.main_backend.url')`) | `POST` | `/sso/company/synchronize` | Sync company | `app/Services/MainBackendSyncService.php` |
| Main backend | `POST` | `/sso/user/synchronize` | Sync user | `MainBackendSyncService.php` |
| Main backend | `GET` | `/sso/user/{externalSsoUserId}/api-keys` | List API keys | `MainBackendSyncService.php` |
| Main backend | `POST` | `/sso/user/{externalSsoUserId}/api-keys` | Create API key | `MainBackendSyncService.php` |
| Service manager (`SERVICE_MANAGER_URL`) | `POST` | `/api/esm-user-create` | Create ESM user (`X-API-KEY`) | `app/Services/EsmUserService.php` |
| SSO server (`settings.sso` JSON: `sso_server_url` + path fields) | `POST` (form) | `{sso_server_url}{ep_oauth_tokens}` | Exchange auth code for tokens | `app/Http/Controllers/SSO/AuthController.php` |
| SSO server | `GET` | `{sso_server_url}{ep_user}` (INFERRED path name from usage) | Fetch user with Bearer token | `AuthController`, `app/Http/Middleware/SsoTokenToSession.php`, `app/Http/Controllers/Client/LoginController.php`, etc. |
| SSO server | `POST` | `{sso_server_url}{ep_verify_token_body from AppHelper}` | Validate Bearer token / user | `app/Helpers/AppHelper.php` (`getTokenStatus`), `app/Http/Middleware/ValidateAPIToken.php` |
| SSO server | `POST` | `{sso_server_url}/api/sso/user/update` | Update SSO user profile (best-effort) | `app/Services/SsoSignupCompletionService.php` |
| SSO server | `POST` | URLs built from admin settings / registration flows | User registration rollback and related | `app/Http/Controllers/Client/RegisterController.php`, `VendorController.php`, `InternalController.php`, `ForgotPasswordController.php`, etc. |
| External IdP | `POST` | URL from import/settings | `ImportController` IDP map flow | `app/Http/Controllers/Admin/Import/ImportController.php` |
| Stripe | SDK (HTTPS to Stripe) | Cartalyst Stripe API | Payments, credits | `app/Http/Controllers/Shop/CartController.php`, `Shop/CreditsController.php` |

SSO path segments (`ep_authorize`, `ep_oauth_tokens`, `client_callback`, etc.) are **not** in `.env`. They live in the **`settings` database table**, column **`sso`**, as JSON (`settings.sso`). Values are edited via the **admin Settings (SSO)** UI (`app/Http/Controllers/Admin/Settings/SettingsController.php`). Per-deployment URLs and client secrets differ by environment; the schema is defined by what the code reads from that JSON (see `AuthController`, `getAppSettings()`, `getTokenStatus()`).

## 6. Database / Models / Tables

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| `users` | Subscribers, admins, IdP linkage (`idp_id`), company | Read / Write | `app/Models/User.php`, migrations |
| `companies` | Organizations | Read / Write | `app/Models/CompanyModel.php`, `database/migrations/2024_11_26_173233_create_companies_table.php` |
| `subscriptions` | Plans/products | Read / Write | `app/Models/Admin/SubscriptionModel.php` |
| `subscription_attributes` | Plan attributes (e.g. limits) | Read / Write | migrations, used across checkout/API |
| `orders` | Orders | Read / Write | `app/Models/Client/OrdersModel.php` |
| `transactions` | Payments / transactions | Read / Write | `app/Models/Client/TransactionsModel.php` |
| `credits` | AI / service credits balance | Read / Write | `app/Models/Client/CreditsModel.php` |
| `credit_hold_amounts` | Credit holds for API usage | Read / Write | `app/Models/Client/CreditHoldAmount.php` |
| `addons` | Add-ons | Read / Write | `app/Models/Admin/AddonModel.php` |
| `settings` | SSO JSON, Stripe/payment JSON, app settings | Read / Write | `app/Models/Admin/SettingsModel.php` |
| `payment_settings` | Payment gateway row(s) used by `stripeSettings()` | Read / Write | `database/migrations/2024_11_27_063348_create_payment_settings_table.php`, `app/Helpers/AppHelper.php` |
| `rtl_entities` | RtL licensing entities | Read / Write | migrations |
| `users_verify` | Email verification tokens | Read / Write | `app/Models/UserVerify.php` |
| `notifications` | User notifications | Read / Write | `app/Models/NotificationModel.php` |
| `user_apps` | Client apps metadata | Read / Write | `app/Models/Client/AppsModel.php` |
| Spatie `permissions` / `roles` / pivots | RBAC | Read / Write | `database/migrations/2024_02_11_103109_create_permission_tables.php` |
| `jobs` / `failed_jobs` | Queue tables (if async queue used) | Varies | migrations |

## 7. Jobs / Queues / Cron / Workers

| Name | Type | Purpose | Source file |
|---|---|---|---|
| `subscriptions:status` | Scheduled command (every minute) | Subscription/order status maintenance | `app/Console/Commands/SubscriptionStatusJob.php`, `app/Console/Kernel.php` |
| `subscriptions:alert` | Scheduled command (daily 09:15) | Early alert emails | `app/Console/Commands/EarlyAlertJob.php`, `app/Console/Kernel.php` |
| `VerifyEmailJob`, `WelcomeMailJob`, `SubscriptionMailJob`, `CancelSubscriptionMailJob`, `CancelMembershipMailJob`, `EarlyAlertMailJob`, `ResetSubscriptionMailJob` | Queue jobs (driver from `QUEUE_CONNECTION`) | Email side effects | `app/Jobs/*.php` |

Default `.env.example` uses `QUEUE_CONNECTION=sync`, so jobs often run inline unless overridden in deployment.

## 8. Configuration & Environment

Integration and runtime settings are primarily **Laravel `.env`** (copy from `.env.example`) plus **database-backed `settings`** and **`payment_settings`** for SSO and Stripe display/keys in admin.

### Primary: `.env` + `config/*.php`

Values are read via `env()` and `config()`; service URLs are centralized in `config/services.php`.

| Key / Name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `APP_URL`, `APP_KEY`, `APP_DEBUG`, `APP_ENV` | App identity and debugging | `.env.example`, standard Laravel |
| `DB_*` | Database connection | `.env.example` |
| `MAIN_BACKEND_URL`, `MAIN_BACKEND_API_KEY` | Main backend base URL and `API-Key` header | `.env.example`, `config/services.php`, `MainBackendSyncService.php` |
| `SERVICE_MANAGER_URL`, `SERVICE_MANAGER_API_KEY` | Service-manager base URL and `X-API-KEY` | `.env.example`, `config/services.php`, `EsmUserService.php` |
| `MAIN_FRONTEND_URL` | Redirects / links to main frontend | `.env.example`, `config/app.php`, `OrganizationController`, `CartController`, `RegisterController` |
| `SSO_SIGNUP_API_KEY` | Shared secret for `/api/sso/*` signup routes | `.env.example`, `config/services.php`, `ValidateSsoSignupApiKey.php` |
| `OAUTH_CLIENT_BOOTSTRAP_SECRET` | Validates `oauth_chain` query on SSO login | `.env.example`, `AuthController.php` |
| `WHITELISTED_IPS` | IP allowlist (credits mutations) | `.env.example`, whitelist middleware |
| `API_KEY` | Header `API-Key` fallback when no Bearer token (credits routes) | `.env.example`, `TokenOrApiKey.php` |
| `MAIL_*`, `MAIL_SUPPORT_URL` | Mail transport and support link in emails | `.env.example` |
| `AWS_*` | Optional object storage | `.env.example` |

### Other / Secondary

| Variable / Key | Purpose | Used In |
|---|---|---|
| `settings.sso` | **`settings` table**, `sso` column (JSON): `sso_server_url`, OAuth paths (`ep_*`), client ids/secrets | `AuthController`, `getAppSettings()`, admin `SettingsController` |
| `payment_settings.stripe` (DB JSON) | Stripe keys, environment, feature flags | `stripeSettings()` in `app/Helpers/AppHelper.php`, `SettingsController` |
| `config/services.php` → `stripe` | Third-party block (prefer DB-backed Stripe for cart in practice) | `config/services.php` |

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| SSO / IdP server | Auth API + OAuth UI | Login, token validation, user APIs, signup updates | `AuthController`, `ValidateAPIToken`, `SsoSignupCompletionService`, DB `settings.sso` |
| Main backend | HTTP API | Company/user sync, subscription user API keys | `MainBackendSyncService`, `config/services.php` |
| Service manager | HTTP API | `esm-user-create` | `EsmUserService.php` |
| Main frontend | Web app / URL | Post-login redirects, onboarding URLs | `MAIN_FRONTEND_URL`, controllers |
| MySQL / MariaDB | Database | All domain data | migrations, `.env.example` |
| Stripe | Payment API | Checkout and credits | `CartController`, `CreditsController`, Cartalyst |
| Mail transport | SMTP / etc. | Transactional email | Laravel mail, jobs |

## 10. Main Flows

### Flow: SSO web login

1. User hits `/sso/login`; app reads `settings.sso` and redirects to SSO authorize URL with `client_id`, `redirect_url`, `state`.
2. SSO redirects to `/callback` with `code`; app exchanges code at `{sso_server_url}{ep_oauth_tokens}` and stores tokens in session.
3. Session is established; `SsoTokenToSession` can restore session from `_em_id` cookie for parity with other apps.

### Flow: SSO-initiated signup API

1. SSO calls `POST /api/sso/complete-signup` with `X-SSO-Signup-Api-Key`.
2. `SsoSignupCompletionService` creates company and user, assigns role, syncs to main backend (`MainBackendSyncService`), queues welcome/verify emails.

### Flow: Subscription checkout and main-backend API keys

1. User uses web cart/checkout (`CartController` and related) with Stripe as configured in DB.
2. On success, subscription attributes exist; `MainBackendSyncService::createUserApiKey` can create keys on main backend for the IdP user.

### Flow: Credits API (service callers)

1. Caller uses `Authorization: Bearer` (SSO access token) or `API-Key` matching `API_KEY`, plus IP whitelist for hold/release/charge.
2. `CreditsController` adjusts `credits` / `credit_hold_amounts` and may use Stripe settings for monetary conversion where applicable.

### Flow: Scheduled maintenance

1. `subscriptions:status` runs every minute; `subscriptions:alert` runs daily for early alerts.

## 11. Things Other Repos Depend On

- **Stable JSON routes under `/api`**: SSO signup (`/api/sso/complete-signup`, `/api/sso/email-verification-status`), subscription lookup (`/api/user/subscription`, `/api/open/*`), credits (`/api/credits/*`).
- **SSO contracts**: Shared `SSO_SIGNUP_API_KEY` header; Bearer tokens validated via SSO endpoints configured in subscription `settings.sso`.
- **Main-backend compatibility**: Expected paths `/sso/company/synchronize`, `/sso/user/synchronize`, `/sso/user/{id}/api-keys` with `API-Key` header.
- **Service-manager contract**: `POST /api/esm-user-create` with `X-API-KEY`.
- **Session / cookie behavior**: `_em_id` cookie and `/sso/session/clear` for logout alignment with main-frontend/service-manager (per middleware comments).
- **Shared OAuth bootstrap**: `oauth_chain` query validated with `OAUTH_CLIENT_BOOTSTRAP_SECRET` when present.

## 12. Unknowns / Needs Confirmation

- Exhaustive list of JSON keys under `settings.sso` for each environment (code references many `ep_*` and client fields; production payloads should match SSO product docs).
- Whether any clients still rely on legacy `api_token` style auth vs SSO-only APIs (`config/auth.php` uses `token` driver for `api` guard).
- Full list of external `Http::post` registration/rollback URLs used in vendor/register flows (built from SSO/`settings` patterns).
- Production queue driver and cron entry for `php artisan schedule:run`.
