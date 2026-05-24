# Create or Update Repo Profile Prompt

Use this prompt when analyzing **any** repository to create or refresh `docs/repo-profile.md`.

The example outputs below are structural only; adapt names and mechanisms to whatever the scanned repo actually uses.

---

You are analyzing one repository.

Your task is to create or update:

`docs/repo-profile.md`

The goal is to help humans and AI agents quickly understand what this repo does, what it exposes, what it calls, and how it connects to other repos/services.

This repo may use any language or framework, including Golang, Python, React, Remix, Laravel, Node.js, PHP, or others.

If `docs/repo-profile.md` already exists, read it first and update it carefully.

If `docs/repo-profile.md` does not exist, create it from scratch.

---

## Repo-specific: `emobiq-migration`

For the **`emobiq-migration`** repository, treat **`docs/erd.md`** as a sibling of **`docs/repo-profile.md`** (both live under `docs/`). When creating or updating the repo profile, also create or update **`docs/erd.md`** as needed so ERD/schema documentation stays aligned with what you found in the codebase—follow any conventions or templates already present in that repo.

---

## What to do

Scan the repository and generate or update a simple factual document.

Focus on:

1. What this repo is for
2. Main language/framework/runtime
3. Important folders/files
4. HTTP/API endpoints this repo exposes
5. HTTP/API endpoints this repo calls
6. Databases/tables/models/entities used
7. Background jobs, cron jobs, queues, workers, or scheduled tasks
8. **Configuration & environment for integrations** — Identify **where** settings live for this codebase (examples: `.env*`, `config.js` / `config.ts`, YAML, Helm values, Terraform, vendor dashboards, secrets managers). Determine which mechanism is primary for service URLs, auth, toggles, and keys. Inspect imports and committed templates such as `.env.example`, `*.sample`, `appsettings*.json`. Do **not** assume every repo is `.env`-first; document what *this* repo actually does. Summarize integration-relevant keys only; cite the canonical sample file instead of copying huge blobs.
9. Other services/repos this repo likely depends on
10. Main business flows handled by this repo
11. Contracts other repos rely on (`postMessage`, webhooks, shared types, stable routes, etc.)

---

## Update Rules

If `docs/repo-profile.md` already exists:

1. Preserve the existing structure and useful wording.
2. Update only sections that are outdated, missing, or incorrect.
3. Remove or correct claims that are no longer supported by the code.
4. Update the top metadata:
   - `Last updated: YYYY-MM-DD`
   - `Confidence: High / Medium / Low`
5. Do not rewrite the whole document unnecessarily.

### Important Section Naming Rule

If section 11 is currently named:

```md
## 11. Things Other Repos Depend On
```

keep that exact heading.

Do not rename it back to:

```md
## 11. Things Other Repos May Depend On
```

---

## Files/folders to check first

Look for relevant files such as:

```txt
README.md
AGENTS.md
.env.example / .env.sample
config.js / config.ts / config.*
appsettings*.json
settings.*
docker-compose.yml
Dockerfile
package.json
composer.json
go.mod
pyproject.toml
requirements.txt
routes/
api/
app/
src/
server/
controllers/
handlers/
services/
clients/
repositories/
models/
database/
migrations/
jobs/
workers/
cron/
queues/
config/
```

Do not scan blindly forever. Prioritize files that reveal routes, API clients, services, database models, and configuration.

---

## When To Update

Update the doc when changes affect:

- exposed endpoints
- outbound API calls
- auth/session behavior
- config/environment keys
- service dependencies
- database/models/tables
- jobs/cron/workers
- iframe/postMessage contracts
- shared behavior other repos rely on
- main business/data flows

Skip updates for small UI-only changes, copy changes, tests-only changes, or internal refactors with no cross-repo impact.

---

## Output Format

Create or update `docs/repo-profile.md` with this structure:

```md
# Repo Profile: <repo-name>

Last updated: <YYYY-MM-DD>
Confidence: High / Medium / Low

## 1. Purpose

Short explanation of what this repo does.

## 2. Tech Stack

- Language:
- Framework:
- Runtime:
- Package manager:
- Database:
- Other important dependencies:

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `path/example` | What it is used for |

## 4. Exposed Endpoints

Endpoints provided by this repo.

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/example` | Example purpose | `path/to/file` | Yes / No / Unknown |

If this repo does not expose HTTP endpoints, write:

This repo does not appear to expose HTTP endpoints.

## 5. Outbound API Calls

External/internal endpoints this repo calls.

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| `service-name` | `POST` | `/example` | Example purpose | `path/to/file` |

Include calls found through:

- HTTP clients
- fetch/axios
- requests/httpx
- Guzzle
- Go http client
- SDK clients
- config- or env-based base URLs

If unclear, mark as `NEEDS CONFIRMATION`.

## 6. Database / Models / Tables

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| `users` | Example purpose | Read / Write / Unknown | `path/to/file` |

Include migrations, models, ORM entities, repositories, and raw SQL if found.

## 7. Jobs / Queues / Cron / Workers

| Name | Type | Purpose | Source file |
|---|---|---|---|
| `job-name` | Cron / Queue / Worker | Example purpose | `path/to/file` |

If none found, write:

No jobs, queues, cron tasks, or workers were found.

## 8. Configuration & Environment

Describe where integration/runtime settings live in this repo.

Use one or both subsections as appropriate.

### Primary: `<mechanism>`

Brief sentence on why this is primary, including how the app imports or reads values.

| Key / Name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `EXAMPLE_KEY` | Example purpose | `path/to/sample`, `path/to/consumer` |

### Other / Secondary

| Variable / Key | Purpose | Used In |
|---|---|---|
| `EXAMPLE_KEY` | Example purpose | `path/to/file` |

Rules:

- Never paste real secrets or credential values.
- If the primary file is huge, summarize cross-repo-critical URLs, flags, and API key names only.
- Point readers to the committed template path for the full inventory.

## 9. Service Dependencies

Services, repos, databases, queues, storage, or external systems this repo appears to depend on.

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| `service-name` | API / Database / Queue / Storage / Auth | Purpose | `path/to/file` |

## 10. Main Flows

Describe the most important flows handled by this repo.

### Flow: <flow name>

1. Step one
2. Step two
3. Step three

Mention files involved where helpful.

## 11. Things Other Repos Depend On

List endpoints, data, contracts, events, or behavior from this repo that other repos likely rely on.

- Example contract or behavior

## 12. Unknowns / Needs Confirmation

List anything important that could not be confirmed from the code.

- NEEDS CONFIRMATION: Example unknown
```

---

## Rules

- Be factual.
- Do not invent endpoints, services, tables, configs, or flows.
- If something is inferred, label it as `INFERRED`.
- If something is unclear, label it as `NEEDS CONFIRMATION`.
- Include source file paths as evidence wherever possible.
- Keep the document useful and concise.
- Prefer an incomplete but accurate document over a complete but guessed one.
- Do not include secrets, tokens, passwords, or private credentials.
- Do not document every tiny helper function.
- Focus on cross-repo understanding.
- Skip updates if the current code changes do not affect cross-repo understanding.

---

## Final Task

Create or update:

- `docs/repo-profile.md`

For **`emobiq-migration`**, also create or update **`docs/erd.md`** (same `docs/` folder as the repo profile).

After editing, reply only with a short summary and any remaining `NEEDS CONFIRMATION` items.