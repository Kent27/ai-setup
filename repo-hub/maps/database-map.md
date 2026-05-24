# Database Map

## Overview

This map aggregates **databases and other durable state** described in `repo-hub/repo-profiles/*.md` and relevant `repo-hub/flows/*.md`. Each **top-level section** is one **store** (database or non-SQL state); **tables** for that store sit directly under it.

It is **not** a full DDL inventory: detailed MariaDB/PostgreSQL inventories live in **`repo-profiles/emobiq-migration.md`** (and **`docs/erd.md`** in the **emobiq-migration** repo per that profile).

---

## Platform MariaDB

**Type:** MariaDB  
**Used by:** `emobiq-main-backend`, `emobiq-v5-platform`, `emobiq-migration`  
**Purpose:** Canonical platform data (apps, users, OAuth, builds, API keys, etc.). **`emobiq-migration`** applies schema; **`emobiq-main-backend`** is the primary documented **runtime API** owner.  
**Confidence:** High  

### Tables

| Table / model | Used by | Purpose | Confidence |
|---|---|---|---|
| `users` | `emobiq-main-backend`, `emobiq-v5-platform`, `emobiq-migration` | Platform users (`external_sso_user_id` etc. per flows). Schema via `emobiq-migration`. | High |
| `companies` | `emobiq-main-backend`, `emobiq-migration` | Organizations. | High |
| `applications` | `emobiq-main-backend`, `emobiq-v5-platform`, `emobiq-migration` | Application records. | High |
| `applications_type` | `emobiq-main-backend`, `emobiq-migration` | Application typing. | High |
| `api_keys` | `emobiq-main-backend`, `orangekloud-subscription` (via API), `emobiq-v5-platform` (rate-limit path) | Connector keys + limits (`daily_limit`, `burst_limit_per_minute`, `is_active`). | High |
| `oauth_access_tokens`, `oauth_refresh_tokens`, `oauth_clients`, `oauth_scopes` | `emobiq-main-backend`, `emobiq-migration` | OAuth2 tokens/clients on platform DB (per migration inventory). | High |
| `audits`, `licenses`, `roles` | `emobiq-main-backend`, `emobiq-migration` | Audit, licensing, roles. | High |
| `build`, `build_types`, `publishings`, `publishing_channels`, `platform` | `emobiq-main-backend`, `emobiq-migration` | Build/publish pipeline. | High |
| `banners`, `packages`, `cookies` | `emobiq-main-backend`, `emobiq-migration` | Banners, packages, stored cookies. | High |
| `external_apis`, `external_api_domains` | `emobiq-main-backend`, `emobiq-migration` | External API registry. | High |
| `support_tickets`, `support_ticket_activities`, `support_ticket_attachments`, `support_ticket_notes`, `support_ticket_watchlists` | `emobiq-main-backend`, `emobiq-migration` | Support ticketing. | High |
| `mcp_sessions` | `emobiq-main-backend`, `emobiq-migration` | MCP session persistence. | High |
| `custom_uis` | `emobiq-main-backend` | Custom UI payloads. | Medium |
| `applications`, `users`, `packages`, `dv_*` (legacy Q entities) | `emobiq-v5-platform` | Legacy **PHP** platform MariaDB metadata (v5 profile). Overlapping names with Go platform tables; **NEEDS CONFIRMATION** same DB instance as main-backend in all envs. | Medium |

---

## Subscription app database (MySQL / MariaDB)

**Type:** MySQL / MariaDB (`DB_CONNECTION=mysql` per profile)  
**Used by:** `orangekloud-subscription`  
**Purpose:** Plans, orders, credits, companies, users, `settings` / SSO JSON, Spatie permission tables, queue tables if async.  
**Confidence:** High  

### Tables

| Table / model | Used by | Purpose | Confidence |
|---|---|---|---|
| `users` | `orangekloud-subscription`, flows linking SSO | Subscribers/admins; `idp_id` links SSO user id. | High |
| `companies` | `orangekloud-subscription`, `emobiq-main-backend` (sync via API) | Subscription-side orgs. | High |
| `subscriptions`, `subscription_attributes` | `orangekloud-subscription`, license flows | Plans + attributes (e.g. connector limits). | High |
| `orders` | `orangekloud-subscription` | Checkout; active sub via `order_status` per flow. | High |
| `credits`, `credit_hold_amounts` | `orangekloud-subscription`, credit API callers | Credits / holds. | High |
| `settings` (`sso` JSON column) | `orangekloud-subscription` | SSO OAuth paths/URLs for web login. | High |
| `transactions`, `addons`, `payment_settings`, `rtl_entities`, `users_verify`, `notifications`, `user_apps` | `orangekloud-subscription` | Payments, add-ons, verification, etc. | High |
| `jobs`, `failed_jobs` | `orangekloud-subscription`, Laravel queue | Async mail/jobs when `QUEUE_CONNECTION` not `sync`. | Medium |
| Spatie `permissions`, `roles`, pivots | `orangekloud-subscription` | RBAC. | High |

---

## SSO app database (MySQL / MariaDB)

**Type:** MySQL / MariaDB (`DB_CONNECTION=mysql` per profile)  
**Used by:** `orangekloud-sso`  
**Purpose:** IdP users, Laravel Passport tokens/clients, password resets, app `settings`.  
**Confidence:** High  

### Tables

| Table / model | Used by | Purpose | Confidence |
|---|---|---|---|
| `users` | `orangekloud-sso` | IdP credentials; admin guard shares table per profile. | High |
| `oauth_clients`, `oauth_access_tokens`, `oauth_refresh_tokens`, `oauth_auth_codes`, `oauth_personal_access_clients` | `orangekloud-sso`, OAuth clients | Laravel Passport. | High |
| `password_resets` | `orangekloud-sso` | Password reset tokens. | High |
| `settings` | `orangekloud-sso` | App metadata (`eps_url` etc.). | Medium |
| `personal_access_tokens` | NEEDS CONFIRMATION | Migration present; usage unclear per profile. | Low |
| `failed_jobs` | NEEDS CONFIRMATION | Schema present per profile. | Low |

---

## ESM PostgreSQL

**Type:** PostgreSQL  
**Used by:** `emobiq-service-manager`, `emobiq-migration` (schema)  
**Purpose:** Service Manager package/plugin metadata (`esm_*`). **`emobiq-migration`** registers ESM schema migrations.  
**Confidence:** High  

### Tables

| Table / model | Used by | Purpose | Confidence |
|---|---|---|---|
| `esm_lib`, `esm_lib_deps` | `emobiq-service-manager`, `emobiq-main-backend`, `orangekloud-subscription`, `emobiq-migration` | Libraries / deps. | High |
| `esm_package`, `esm_package_contribs`, `esm_package_deps`, `esm_package_editlib`, `esm_package_libdeps`, `esm_package_plugins`, `esm_package_tags` | `emobiq-service-manager`, `emobiq-main-backend`, `orangekloud-subscription`, `emobiq-migration` | Package graph / metadata. | High |
| `esm_user` | `emobiq-service-manager`, `emobiq-main-backend`, `orangekloud-subscription`, `emobiq-migration` | ESM users; `external_sso_user_id` per profile. | High |
| `user_plugin_actions` | `emobiq-service-manager`, `emobiq-main-backend`, `orangekloud-subscription`, `emobiq-migration` | Per-user plugin actions. | High |
| `migrations` | `emobiq-migration`, `emobiq-service-manager` (runtime) | ESM-side migration tracking. | High |

---

## Autopilot PostgreSQL

**Type:** PostgreSQL  
**Used by:** `autopilot-ai` (runtime); `emobiq-migration` (**partial** schema)  
**Purpose:** AI projects, runs, documents, chat, costs, etc. **`emobiq-migration`** warns Autopilot migrations may **lag** production.  
**Confidence:** High  

### Tables

| Table / model | Used by | Purpose | Confidence |
|---|---|---|---|
| `project`, `project_details`, `run`, `progress`, `document`, `cost`, `cost_details`, `chat_history`, `human_input`, `supabase_credentials`, `progress_notes`, `plugins` | `autopilot-ai`, `autopilot-debug-agent` (HTTP only), `emobiq-migration` (partial snapshot) | Core AI workspace. | High |
| `templates`, `template_bookmarks`, `plugin_credentials` | `autopilot-ai` | Templates / bookmarks / plugin secrets. | High |

---

## Per-app SQLite (`data.db` pattern)

**Type:** SQLite files on disk  
**Used by:** `emobiq-main-backend`, `emobiq-v5-platform`, `emobiq-migration`  
**Purpose:** App design content, pages/snippets, staging; paths from each repo’s `database` / API config.  
**Confidence:** High  

### Tables / surfaces

| Table / surface | Used by | Purpose | Confidence |
|---|---|---|---|
| `eas_page` and related app DB tables | `emobiq-v5-platform` | v5 app SQLite pages. | High |
| Client/server models (e.g. `pages`, `snippets`, custom styles, server-side app DB tables) | `emobiq-main-backend`, `emobiq-migration` | Packaged in compile tarball (`data.db`). | High |

Full per-app table lists: **`repo-profiles/emobiq-migration.md`** SQLite schema sections.

---

## Redis (optional)

**Type:** Redis  
**Used by:** `emobiq-main-backend`  
**Purpose:** Connector raw V2 **rate limits** when `database.json` sets `redis.connector_raw_rate_limit`.  
**Confidence:** High  

### Documented usage (not relational tables)

| Key / pattern | Used by | Purpose | TTL / lifecycle | Confidence |
|---|---|---|---|---|
| Rate-limit counters for `api_keys` (**exact key pattern NEEDS CONFIRMATION**) | `emobiq-main-backend` | `daily_limit` / `burst_limit_per_minute` | Flow: daily midnight SGT + short burst TTL | Medium |

---

## MongoDB (optional / tooling)

**Type:** MongoDB  
**Used by:** `emobiq-socket`, `emobiq-compiler`, `emobiq-builder`  
**Purpose:** Profiles: **HTTP servers** usually **do not** `Initialize()` DB; migration CLI / scaffold may use collections.  
**Confidence:** Medium  

### Collections

| Collection / area | Used by | Notes | Confidence |
|---|---|---|---|
| `migrations` (and generic CRUD) | Migration CLIs where wired | Runtime WebSocket/build paths often skip Mongo. | Medium |
| `users` | Unregistered / dead paths in some repos | Compiler/builder profiles say not used by live HTTP. | Medium |

---

## In-memory state (not durable DB)

**Type:** In-process memory  
**Used by:** `emobiq-socket`  
**Purpose:** WebSocket hub channels/clients (no MariaDB/Redis on documented live path).  
**Confidence:** High  

*(No tables.)*

---

## Filesystem & artifact storage

**Type:** Local (or mounted) filesystem  
**Used by:** `emobiq-storage`, `emobiq-compiler`, `emobiq-builder`, `autopilot-debug-agent`, `emobiq-main-backend` (paths)  
**Confidence:** High  

### Documented paths / areas

| Path / area | Used by | Purpose | TTL / lifecycle | Confidence |
|---|---|---|---|---|
| Repo root static tree (`app/`, `uploads/`, etc.) | `emobiq-storage` | Served artifacts | Ops | High |
| `backend/storage/tmp`, `lock`, `files` | `emobiq-compiler` | Compile workspace, locks, templates | Lock cleanup; `deleteTmpFile` | High |
| `backend/storage/scp/…`, lock dirs | `emobiq-builder` | Staged builds | Stale-lock recovery | High |
| `LOG_DIRECTORY` + `YYYY-MM-DD.log` | `autopilot-debug-agent` | Log retention | `LOG_RETENTION_DAYS` | High |
| `projectStoragePath`, SQLite paths in `database.json` | `emobiq-main-backend` | Artifacts + `data.db` layout | Tmp cron (`storage/tmp/bcapi`) | High |

---

## Data ownership notes

- **Schema vs runtime:** **`emobiq-migration`** versioned schema + data migrations for platform MariaDB, ESM/Autopilot PostgreSQL, SQLite templates; **`emobiq-main-backend`** owns routine **HTTP** access to platform MariaDB/SQLite.
- **Separate `users` tables:** **`orangekloud-sso`** and **`orangekloud-subscription`** each have a **`users`** table on **their** app DB—not the same physical row (**NEEDS CONFIRMATION** for rare shared-DB setups).
- **Autopilot:** Runtime source of truth **`autopilot-ai`**; **`emobiq-migration`** Autopilot migrations may trail production.
- **ERD:** **`repo-profiles/emobiq-migration.md`** → **`docs/erd.md`** in the **emobiq-migration** repo.

---

## Known gaps / needs confirmation

- Whether **platform MariaDB** for **`emobiq-v5-platform`** and **`emobiq-main-backend`** is one **shared** instance everywhere or split by era/tenant.
- **Exact Redis key naming** for connector rate limits.
- **MongoDB** required in production for compiler/builder/socket vs migration-only.
- Full **SQLite** column/table list without duplicating **`emobiq-migration`**.
- **DynamoDB** / other cloud tables only via **client-frontend** credentials—not hub-enumerated.
