# Repo Profile: emobiq-data-migration

Last updated: 2026-05-07
Confidence: High

## 1. Purpose

CLI-focused Go service that applies **database schema migrations**, **default-data seeding**, and **versioned data migrations** across MariaDB (platform), PostgreSQL (**ESM** is maintained here; **Autopilot** schema in this repo may lag your latest definition until migrations are added), and per-application SQLite databases. It also includes a separate **`cmd/data`** helper that copies archive/project directories from a configured source (e.g. legacy v5 layout) to a target storage path. This repo does **not** implement product APIs for client apps; it is used for migrating and aligning database/state with other OrangeKloud backends.

## 2. Tech Stack

- Language: Go (`go 1.18` per [`golang/go.mod`](golang/go.mod); [`golang/README.md`](golang/README.md) mentions newer local toolchain versions)
- Framework: [Gin](https://github.com/gin-gonic/gin) (wired in [`golang/main.go`](golang/main.go), but **no routes registered** in [`golang/route/api.go`](golang/route/api.go))
- Runtime: Native CLI (`go run`) from [`golang/`](golang/) working directory; optional Gin HTTP entrypoint with empty route table
- Package manager: Go modules ([`golang/go.mod`](golang/go.mod))
- Database: MariaDB/MySQL, PostgreSQL (ESM + Autopilot migration targets; Autopilot schema in-repo may be incomplete), SQLite via [GORM](https://gorm.io/) and [gormigrate](https://github.com/go-gormigrate/gormigrate)
- Other important dependencies: `github.com/tidwall/gjson`, `gopkg.in/yaml.v3` (YAML parsing in migrations, not HTTP)

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| [`golang/cmd/migrations/main.go`](golang/cmd/migrations/main.go) | Primary CLI: `<mariadb\|pgsql\|sqlite> <schema\|default-data\|data> ...` |
| [`golang/cmd/data/main.go`](golang/cmd/data/main.go) | Copies configured directories from `dirMigration.source` to `dirMigration.target` |
| [`golang/app/library/migrations/migration.go`](golang/app/library/migrations/migration.go) | Registry and orchestration for schema/default-data/data migrations |
| [`golang/database/database.go`](golang/database/database.go) | GORM connection helpers for mariadb / pgsql / sqlite |
| [`golang/database/migration/schema/`](golang/database/migration/schema/) | Schema migration definitions per engine |
| [`golang/database/migration/data/`](golang/database/migration/data/) | Data migration steps per engine |
| [`golang/app/model/`](golang/app/model/) | GORM models (MariaDB, PostgreSQL namespaces, SQLite server/client) |
| [`golang/config/application.json.sample`](golang/config/application.json.sample) | Application paths, migration CSV paths pattern |
| [`golang/config/database.json.sample`](golang/config/database.json.sample) | DB connection blocks and SQLite path templates |
| [`golang/route/api.go`](golang/route/api.go) | HTTP route registration (currently empty) |
| [`golang/main.go`](golang/main.go) | Gin server bootstrap (logs to `log/gin/gin.log`, port from config) |

## 4. Exposed Endpoints

This repo does not appear to expose HTTP endpoints for its primary use case: [`golang/route/api.go`](golang/route/api.go) registers no routes. [`golang/main.go`](golang/main.go) can start Gin on `application.port`, but only default Gin middleware applies—no application paths are defined in code.

## 5. Outbound API Calls

No outbound HTTP clients (`net/http`, SDKs, etc.) were found in a scan of [`golang/**/*.go`](golang/). Integration with “other services” is via **filesystem paths** (storage, CSV exports) and **database connections**, not HTTP APIs in this codebase.

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| — | — | — | No outbound HTTP calls identified | — |

## 6. Database / Models / Tables

> **Scope:** The MariaDB and PostgreSQL tables below are inferred from `AutoMigrate` / `DropTable` in [`golang/database/migration/schema/`](golang/database/migration/schema/). For column types and constraints, open the linked migration file and GORM model.

**Mermaid ERDs** (relationship overview, not full DDL): [`docs/erd.md`](erd.md).

**MariaDB** and **PostgreSQL (ESM)** tables below match what this repository currently defines via **GORM models** and **gormigrate** under [`golang/database/migration/schema/`](golang/database/migration/schema/). **Autopilot** is documented only as **what exists in this repo today**—it is **not** claimed to be the latest full schema until new migrations are merged here.

Below are **table name inventories** (not full `CREATE TABLE` DDL—generate that by running migrations or inspecting those sources).

### MariaDB — platform tables (`database.mariadb` in config)

Managed under [`golang/database/migration/schema/mariadb/`](golang/database/migration/schema/mariadb/). Base bootstrap plus later migrations add columns (same logical tables).

| Table | Introduced / notable migration | Model package |
|---|---|---|
| `applications` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/application.go`](golang/app/model/application.go) |
| `applications_type` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/applications_type.go`](golang/app/model/applications_type.go) |
| `audits` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/audit.go`](golang/app/model/audit.go) |
| `users` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/user.go`](golang/app/model/user.go) |
| `roles` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/role.go`](golang/app/model/role.go) |
| `companies` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/company.go`](golang/app/model/company.go) |
| `licenses` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/license.go`](golang/app/model/license.go) |
| `oauth_scopes` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/oauth_scope.go`](golang/app/model/oauth_scope.go) |
| `oauth_clients` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/oauth_client.go`](golang/app/model/oauth_client.go) |
| `oauth_access_tokens` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/oauth_access_token.go`](golang/app/model/oauth_access_token.go) |
| `oauth_refresh_tokens` | [`schema.go`](golang/database/migration/schema/mariadb/schema.go) | [`app/model/oauth_refresh_token.go`](golang/app/model/oauth_refresh_token.go) |
| `platform` | [`create_table_build_publishing.go`](golang/database/migration/schema/mariadb/create_table_build_publishing.go) | [`app/model/platform.go`](golang/app/model/platform.go) |
| `build_types` | [`create_table_build_publishing.go`](golang/database/migration/schema/mariadb/create_table_build_publishing.go) | [`app/model/build_type.go`](golang/app/model/build_type.go) |
| `publishing_channels` | [`create_table_build_publishing.go`](golang/database/migration/schema/mariadb/create_table_build_publishing.go) | [`app/model/publishing_channel.go`](golang/app/model/publishing_channel.go) |
| `build` | [`create_table_build_publishing.go`](golang/database/migration/schema/mariadb/create_table_build_publishing.go) | [`app/model/build.go`](golang/app/model/build.go) |
| `publishings` | [`create_table_build_publishing.go`](golang/database/migration/schema/mariadb/create_table_build_publishing.go) | [`app/model/publishing.go`](golang/app/model/publishing.go) |
| `banners` | [`create_table_banner.go`](golang/database/migration/schema/mariadb/create_table_banner.go) | [`app/model/banner.go`](golang/app/model/banner.go) |
| `packages` | [`create_table_package.go`](golang/database/migration/schema/mariadb/create_table_package.go) | [`app/model/package.go`](golang/app/model/package.go) |
| `external_apis`, `external_api_domains` | [`create_table_external_apis.go`](golang/database/migration/schema/mariadb/create_table_external_apis.go) | [`app/model/external_api.go`](golang/app/model/external_api.go), [`app/model/external_api_domain.go`](golang/app/model/external_api_domain.go) |
| `cookies` | [`create_table_cookie.go`](golang/database/migration/schema/mariadb/create_table_cookie.go) | [`app/model/cookie.go`](golang/app/model/cookie.go) |
| `support_tickets`, `support_ticket_activities`, `support_ticket_attachments`, `support_ticket_notes` | [`create_table_support_ticket.go`](golang/database/migration/schema/mariadb/create_table_support_ticket.go) | [`app/model/support_ticket.go`](golang/app/model/support_ticket.go) and related `support_ticket_*.go` |
| `support_ticket_watchlists` | Same `AutoMigrate` as support ticket migration | [`app/model/support_ticket_watchlist.go`](golang/app/model/support_ticket_watchlist.go) (default GORM pluralization) |
| `mcp_sessions` | [`create_table_mcp_sessions.go`](golang/database/migration/schema/mariadb/create_table_mcp_sessions.go) | [`app/model/mcp_session.go`](golang/app/model/mcp_session.go) |
| `api_keys` | [`create_table_api_keys.go`](golang/database/migration/schema/mariadb/create_table_api_keys.go) | [`app/model/api_keys.go`](golang/app/model/api_keys.go) |

Additional MariaDB migration files only **alter** existing tables (indexes, FKs, column types, collations); see the [`mariadb/`](golang/database/migration/schema/mariadb/) directory listing.

### PostgreSQL — ESM (`--type=esm`)

Sources include [`base_esm_schema.go`](golang/database/migration/schema/pgsql/base_esm_schema.go) and subsequent `schema/pgsql/*.go` migrations registered for `esm` in [`migration.go`](golang/app/library/migrations/migration.go). ESM models live under [`app/model/pgsql/`](golang/app/model/pgsql/) (structs prefixed `ESM*` and related).

| Table | Notes |
|---|---|
| `esm_lib` | Core ESM library metadata |
| `esm_lib_deps` | Library dependency edges |
| `esm_package` | Packages (+ later migrations add columns) |
| `esm_package_contribs`, `esm_package_deps`, `esm_package_editlib`, `esm_package_libdeps`, `esm_package_plugins`, `esm_package_tags` | Package graph / metadata |
| `esm_user` | ESM users (+ later migrations add columns) |
| `migrations` | ESM-side migration tracking (`pgsql.Migrations`) |
| `user_plugin_actions` | [`create_table_user_plugin_action.go`](golang/database/migration/schema/pgsql/create_table_user_plugin_action.go) |

### PostgreSQL — Autopilot (`--type=autopilot`)

**Status:** Maintainer note — **the latest Autopilot schema has not all been added to this repo yet.** The table list below is only **what [`autopilot_schema.go`](golang/database/migration/schema/pgsql/autopilot_schema.go) currently migrates**; treat it as **partial / stale** relative to production until follow-up schema migrations are committed and registered.

**Snapshot (as committed — incomplete):**

| Table |
|---|
| `project`, `project_details`, `run`, `progress`, `document`, `cost`, `cost_details`, `chat_history`, `human_input`, `supabase_credentials`, `progress_notes`, `plugins` |

**Note:** [`app/model/pgsql/`](golang/app/model/pgsql/) also contains types such as `App`, `User`, `Company`, `Audit`, … used heavily in **data** migrations against PostgreSQL. Those are **not** part of the registered **schema** migration lists for keys `esm` / `autopilot` (the `default` PG schema list is empty). Treat them as migration/runtime models unless a new schema migration registers them.

### SQLite (per-app)

Server/client template DBs: tables created in [`golang/database/migration/schema/sqlite/`](golang/database/migration/schema/sqlite/); models under [`golang/app/model/sqlite/`](golang/app/model/sqlite/). Not duplicated here—see those packages for the full list.

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| (many, engine-specific) | Platform / ESM / Autopilot / per-app SQLite | Write via migrations; read/write in data migration steps | [`golang/app/model/`](golang/app/model/), [`golang/database/migration/`](golang/database/migration/) |

## 7. Jobs / Queues / Cron / Workers

No jobs, queues, cron tasks, or workers were found. Interactive CLI prompts and migration logs are used instead.

## 8. Configuration & Environment

Runtime reads **strict JSON** from [`golang/config/application.json`](golang/config/application.json) and [`golang/config/database.json`](golang/config/database.json) via [`golang/app/helper/config.go`](golang/app/helper/config.go). Committed templates use `#` comments ([`*.sample`](golang/config/application.json.sample)); operators must produce valid JSON for runtime.

### Primary: `config/*.json` + process env

JSON files under [`golang/config/`](golang/config/) are the main mechanism for DB hosts, SQLite path templates, storage paths, and migration-specific file paths. Working directory must be [`golang/`](golang/) so [`helper.DirectoryAppRoot()`](golang/app/helper/directory.go) resolves `./config/*` (non-compiled CLI sets [`helper.EnvAppCompiled = false`](golang/cmd/migrations/main.go)).

| Key / Name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `application.port` | HTTP port for [`golang/main.go`](golang/main.go) Gin server | [`golang/config/application.json.sample`](golang/config/application.json.sample), [`golang/main.go`](golang/main.go) |
| `application.dirMigration` (`source`, `target`, `directories`) | Source/target paths for [`cmd/data`](golang/cmd/data/main.go) copy | [`golang/config/application.json.sample`](golang/config/application.json.sample), [`golang/cmd/data/main.go`](golang/cmd/data/main.go) |
| `application.projectStoragePath` | Base storage for built artifacts / app folders in some MariaDB data migrations | [`golang/config/application.json.sample`](golang/config/application.json.sample), e.g. [`golang/database/migration/data/mariadb/migrate_build_publishing.go`](golang/database/migration/data/mariadb/migrate_build_publishing.go) |
| `application.migration.v9.ssoUserFilePath` / `licenseCompanyFilePath` | CSV inputs for SSO-related data migration | [`golang/config/application.json.sample`](golang/config/application.json.sample), [`golang/database/migration/data/mariadb/migrate_external_sso_subscription.go`](golang/database/migration/data/mariadb/migrate_external_sso_subscription.go) |
| `database.mariadb.default` | MariaDB connection | [`golang/config/database.json.sample`](golang/config/database.json.sample), [`golang/database/database.go`](golang/database/database.go) |
| `database.pgsql` (`default`, `sso`, `esm`, `autopilot`) | PostgreSQL connection profiles | [`golang/config/database.json.sample`](golang/config/database.json.sample), [`golang/database/database.go`](golang/database/database.go) |
| `database.sqlite` (`template_server`, `template_client`, `apps` with `{appid}`) | SQLite template and per-app DB paths | [`golang/config/database.json.sample`](golang/config/database.json.sample), [`golang/database/database.go`](golang/database/database.go) |
| `APP_ENV` | When `dev`, skips confirmation prompts for `default-data` / `data` | [`golang/app/library/migrations/migration.go`](golang/app/library/migrations/migration.go), [`golang/README.md`](golang/README.md) |

### Other / Secondary

| Variable / Key | Purpose | Used In |
|---|---|---|
| Root `Makefile` / root `docker-compose*.yml` | Refer to legacy `backend/` paths | Not authoritative for Go layout ([`AGENTS.md`](AGENTS.md)); prefer [`golang/README.md`](golang/README.md) |

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| MariaDB / MySQL | Database | Platform schema + data migrations | [`golang/database/database.go`](golang/database/database.go), [`golang/config/database.json.sample`](golang/config/database.json.sample) |
| PostgreSQL | Database | ESM schema maintained here; Autopilot schema **partial** until migrations catch up; legacy PG data migrations | [`golang/database/database.go`](golang/database/database.go), §6 |
| SQLite files | Database | Per-application `data.db` migrations | [`golang/database/database.go`](golang/database/database.go) |
| Filesystem storage | Storage | Project archives, binaries, CSV inputs | [`golang/cmd/data/main.go`](golang/cmd/data/main.go), [`golang/config/application.json.sample`](golang/config/application.json.sample) |
| External “main-backend” / v5 layout | Repo / deployment layout | **INFERRED** naming from comments in [`golang/cmd/data/main.go`](golang/cmd/data/main.go) and README migration narrative | Comments and [`golang/README.md`](golang/README.md) |

## 10. Main Flows

### Flow: Run schema migration

1. Operator runs from [`golang/`](golang/): `go run cmd/migrations/main.go <db> schema <up|down> [id] --type=... --all=...`.
2. [`cmd/migrations/main.go`](golang/cmd/migrations/main.go) validates db type, operation, and flags.
3. [`migrations.Migrate`](golang/app/library/migrations/migration.go) selects the migration list (MariaDB list vs PG type `esm`/`autopilot` vs SQLite client/server) and executes via gormigrate.

### Flow: Run default-data or versioned data migration

1. Same CLI with `default-data` or `data [version]`.
2. Unless `APP_ENV=dev`, [`prompt`](golang/app/library/prompt/confirm.go) requires typed confirmation (`yes`).
3. Registered functions run against the configured DB(s); SQLite `--all=true` iterates apps per type.

### Flow: Copy v5 archive directories to storage (`cmd/data`)

1. [`cmd/data/main.go`](golang/cmd/data/main.go) reads `application.dirMigration` from config.
2. For each directory name under `directories`, copies `source/<dir>` → `target/<dir>` if target is effectively empty (guards against overwriting).

## 11. Things Other Repos Depend On

- **Stable migration IDs and ordering**: New migrations must be registered in [`golang/app/library/migrations/migration.go`](golang/app/library/migrations/migration.go); execution order follows slice/map registration ([`golang/app/library/migrations/AGENTS.md`](golang/app/library/migrations/AGENTS.md)).
- **PostgreSQL `--type` contract**: Consumers running PG schema migrations must pass `esm` or `autopilot` (or use the correct registry key); mismatch yields no target list ([`AGENTS.md`](AGENTS.md)).
- **SQLite path layout**: `{appid}` placeholder and template paths in [`golang/config/database.json.sample`](golang/config/database.json.sample) must match how sibling repos lay out `data.db` files under storage.
- **CSV-driven v9 SSO migration**: Paths under `application.migration.v9.*` must exist where operators configure them ([`golang/database/migration/data/mariadb/migrate_external_sso_subscription.go`](golang/database/migration/data/mariadb/migrate_external_sso_subscription.go)).

## 12. Unknowns / Needs Confirmation

- NEEDS CONFIRMATION: Whether [`golang/main.go`](golang/main.go) Gin server is deployed anywhere or only legacy scaffolding.
- NEEDS CONFIRMATION: Exact sibling repo names and deployment topology (“main-backend”, “emobiq-storage”) beyond comments and README narrative.
- Autopilot: **full** PostgreSQL schema vs **what is currently merged** in this repo — align §6 Autopilot snapshot after pending migrations land.
