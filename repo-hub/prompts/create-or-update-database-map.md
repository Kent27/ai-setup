# Create or Update Database Map Prompt

You are updating the repository documentation hub named `repo-hub`.

Target file:

`repo-hub/maps/database-map.md`

If the `repo-hub/maps/` folder does not exist, create it.

If the target file does not exist, create it.

If the target file already exists, update it carefully.

## Source of Truth

Use only information already documented in:

- `repo-hub/repo-profiles/*.md`
- `repo-hub/flows/*.md` if relevant

Do not inspect source code unless explicitly instructed.

## Task

Create or update a system-wide database and state map.

The database map should help developers and AI agents understand:

1. Which databases/state stores exist
2. Which repos/services use each database/state store
3. Which tables/models/collections are touched
4. Which Redis keys, queues, storage buckets, or file-based state are documented
5. Which data ownership or usage details are missing or uncertain

Put **per-table ownership** or **read vs write** only in the **Purpose** (or **Notes**) column when profiles document it—do not add separate Owner or Read/Write columns.

## Update Rules

If `repo-hub/maps/database-map.md` already exists:

- Preserve the **per-store layout**: each database/state system gets its own `##` section, with tables (or keys/paths) nested under `### Tables` (or equivalent)—do not collapse into one global “all tables” table.
- Add missing database/state information from repo profiles and flow docs.
- Correct outdated information when repo profiles or flow docs clearly contradict it.
- Do not remove useful information unless it is clearly obsolete or contradicted.
- Preserve exact table names, model names, field names, Redis keys, queue names, and storage names when available.
- Mark uncertainty clearly.
- Do not invent tables, models, fields, Redis keys, queues, storage buckets, or ownership.

## Required Output Format

**Layout rule:** One **`## <Store name>`** section per **database or distinct state system**. Under each store, put a short **metadata block** (Type, Used by, Purpose, Confidence—paragraph or bullet list), then **`### Tables`** (or **`### Tables / surfaces`**, **`### Documented usage`**, **`### Paths`**) with a **single table** of rows **only for that store**.

Use this structure:

```md
# Database Map

## Overview

Briefly explain what this database map covers and that each `##` section below is one store, with tables (or non-table keys/paths) nested under it.

---

## <Store name> (e.g. Platform MariaDB)

**Type:** …  
**Used by:** …  
**Purpose:** …  
**Confidence:** High / Medium / Low  

### Tables

| Table / model | Used by | Purpose | Confidence |
|---|---|---|---|
| `<table-or-model>` | `<repo-name>` | … (include ownership or R/W here only if documented) | High / Medium / Low |

(For stores with **no** relational tables—Redis, pure filesystem, in-memory—use `### Documented usage` or `### Paths` with the same idea: rows belong only to that store.)

---

(Repeat `## <Store>` + `### Tables` / `### …` for every documented store.)

## Data ownership notes

- …

## Known gaps / needs confirmation

- …
```

**Optional sections** (only if profiles support them):

- Separate **`## Redis`**, **`## MongoDB`**, **`## In-memory state`**, **`## Filesystem & artifact storage`** sections when documented, each with its own subsubsection for keys/collections/paths—not a single flat “non-SQL” table at the end.

## Field Guidance

### Per-store `##` sections

Group by **logical store** (not one global table mixing stores). Typical examples:

- Platform **MariaDB** — include **legacy v5 PHP** MariaDB tables in the **same** `### Tables` block as the Go platform tables (separate rows; do not use a second mini-table or subsection).
- Subscription app **MySQL/MariaDB**
- SSO app **MySQL/MariaDB**
- **ESM** PostgreSQL
- **Autopilot** PostgreSQL
- **Per-app SQLite** (`data.db`)

Non-SQL / infrastructure examples (each can be its own `##`):

- **Redis** (rate limits, caches)
- **MongoDB** (if profiles document it)
- **In-memory** (e.g. WebSocket hub)
- **Filesystem** / artifact paths

### `### Tables` (under each store)

List **only** tables/collections that belong to **that** store. Examples of table names: `users`, `api_keys`, `orders`, `subscriptions`, `subscription_attributes`, `esm_package`, `oauth_access_tokens`. Use ORM/model names only when table names are undocumented.

### Keys, queues, paths

Put **Redis keys**, **Laravel `jobs` tables**, and **filesystem paths** under the **same named store** they belong to (Redis → Redis section; queue tables → often the Subscription DB `### Tables`; file paths → Filesystem section)—**do not** merge unrelated stores into one bottom table.

## Rules

- Treat repo profiles and flow docs as the source of truth.
- Do not invent databases.
- Do not invent tables or models.
- Do not invent fields.
- Do not invent Redis keys, queues, buckets, or file paths.
- If ownership is unclear, write `NEEDS CONFIRMATION` in **Purpose** or **Notes**.
- If usage is inferred, mark it as `INFERRED` in **Purpose** or **Notes`.
- Preserve exact names for tables, fields, models, Redis keys, queues, buckets, and paths when available.
- Keep endpoint details out of this file unless needed to explain data ownership.
- Keep wording practical for future developers and AI coding agents.