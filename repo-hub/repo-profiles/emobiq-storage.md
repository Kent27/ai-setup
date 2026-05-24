# Repo Profile: emobiq-storage

Last updated: 2026-05-06

Confidence: High (runtime and repo layout from committed files); Medium (consumers, production hardening, and full `app/` tree not verified in this scan because `app/*` is gitignored). **INFERRED (stakeholder):** a separate application repo reads and writes project details/files against this storage over HTTP (name and API surface of that repo not in this codebase).

## 1. Purpose

**Product role (stated):** This repository holds **stored project payloads**—metadata and files—that **another repo** is expected to **fetch or send** (e.g. project details, assets). This service’s job is to expose that storage over HTTP and keep a predictable on-disk layout (see `AGENTS.md` for path contracts).

**Implementation:** Node.js/Express serves the repository root as static HTTP content and provides a small set of diagnostic/test routes for request bodies, multipart uploads, cookies, and headers. Workspace guidance also describes it as **artifact-storage** for eMOBIQ-related payloads.

## 2. Tech Stack

- **Language:** JavaScript (CommonJS)
- **Framework:** Express 4.x
- **Runtime:** Node.js (README suggests Node 22.x class; not enforced in repo)
- **Package manager:** npm (`package-lock.json`, lockfileVersion 2)
- **Database:** None in application code (storage is filesystem + static HTTP)
- **Other important dependencies:** `multer` (uploads), `morgan` (access log), `cors`, `body-parser`, `cookie-parser`, `debug`; `firebase-admin` is listed in `package.json` but **not** initialized in current `index.js` (commented example only)

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `docs/repo-profile.md` | Factual repo overview for humans/agents (this document) |
| `index.js` | Express app: middleware, static root, test routes, listens on port **3005** |
| `package.json` / `package-lock.json` | Dependency manifest (no `scripts` block in committed `package.json`) |
| `README.md` | Setup notes (example snippets differ from current `index.js`: sample used port `3999`) |
| `AGENTS.md` | **INFERRED** primary internal doc for storage conventions (`app/`, `plugin-settings/`, ignored paths); may be gitignored—verify in clone |
| `.gitignore` | Ignores `app/*`, `tickets/*`, `node_modules/`, `*.log`, `AGENTS.md` patterns |
| `bitbucket-pipelines.yml` | Bitbucket Pipeline: `dev` branch runs SSH remote `git pull` (clone disabled on pipeline) |
| `access.log` | Written at runtime by Morgan (pattern `*.log` is gitignored) |
| `uploads/` | Multer destination for `/test/binary` and `/test/form-data` (directory created/used at runtime) |
| `app/`, `plugin-settings/`, `user/`, `tickets/` | **INFERRED** payload roots per `AGENTS.md`; largely ignored from git—contract for external path keys |

## 4. Exposed Endpoints

Static hosting: `express.static('.')` serves any file under the repo root (method **GET** for static assets; behavior is standard Express static). **Auth:** None in code (treat as **No** / deployment-specific **NEEDS CONFIRMATION**).

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `POST` | `/test/raw` | Echo JSON body | `index.js` | No |
| `POST` | `/test/binary` | Single file upload via multer | `index.js` | No |
| `POST` | `/test/json` | Echo JSON body | `index.js` | No |
| `POST` | `/test/form-data` | Multipart form + files echo | `index.js` | No |
| `POST` | `/test/urlencoded` | Echo URL-encoded body | `index.js` | No |
| `GET` | `/test/cookies` | Echo cookies (also logs to console) | `index.js` | No |
| `GET` | `/print-headers` | Echo request headers (also logs to console) | `index.js` | No |

## 5. Outbound API Calls

No application-code HTTP clients (`fetch`, `axios`, `http`/`https` requests) were found in scanned `*.js` files.

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| Remote server (SSH) | **INFERRED** shell over SSH | `cd $PATH_FOLDER; git pull origin dev` | Deploy/update working tree on dev host | `bitbucket-pipelines.yml` |

## 6. Database / Models / Tables

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| *(none)* | No ORM, SQL, or in-app DB layer | — | — |

Persistence is filesystem-oriented: static files under the repo tree, multer writes under `uploads/`, and `data.db` is mentioned in `AGENTS.md` as a **legacy schema marker** inside some `app/<id>/` layouts—not accessed by `index.js` in this repo.

## 7. Jobs / Queues / Cron / Workers

No jobs, queues, cron tasks, or workers were found in application code.

CI: Bitbucket Pipeline step on branch `dev` runs a remote deploy script (see `bitbucket-pipelines.yml`).

## 8. Configuration & environment

### Primary: hardcoded values in `index.js` + deployment platform variables

There is **no** committed `.env.example`, `dotenv` usage, or `process.env` reads in scanned code. Runtime integration surface is minimal: fixed port **`3005`**, CORS `origin: true` with credentials, multer `dest: 'uploads/'`, morgan stream to `access.log`.

| Key / name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| *(port)* | HTTP listen port | Hardcoded `const port = 3005` in `index.js` (README example used `3999`—**INFERRED** doc drift) |
| `bitbucket-pipelines.yml` `$SSH`, `$PATH_FOLDER` | Remote deploy target/path | Used in pipeline `script` block **NEEDS CONFIRMATION** (Bitbucket deployment variables) |

### Other (secondary)

| Variable / key | Purpose | Used in |
|---|---|---|
| `firebase-admin` package | Optional push/admin SDK | `package.json`; not active in `index.js` (commented setup) |

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| npm packages (Express ecosystem) | Library | HTTP server, parsing, uploads, logging | `package.json`, `index.js` |
| Local filesystem | Storage | Static files + multer uploads + access log | `index.js` |
| Bitbucket Pipelines + SSH host | CI/Deploy | Pull latest `dev` on server | `bitbucket-pipelines.yml` |
| Companion application repo / services | Consumer **INFERRED** | Get or upload project details and files via HTTP paths under this deployment | Team context + `express.static('.')`; **NEEDS CONFIRMATION** (exact client repo(s) and URL patterns not in scanned code) |

## 10. Main Flows

### Flow: Serve stored artifacts as static files

1. Process starts and binds to port `3005` (`index.js`).
2. A client (**INFERRED:** primary consumer is another repo or service integrating with “projects”) requests a path such as `/app/<id>/…` or other rooted files; `express.static('.')` serves the file from the repo tree when it exists (**NEEDS CONFIRMATION:** authoritative path list lives with the consuming repo or ops).
3. Morgan appends a line to `access.log`.

### Flow: Diagnostic upload / body echo

1. Client sends `POST` to a `/test/*` route with JSON, URL-encoded, multipart, or raw body (or file for binary route).
2. Middleware (`body-parser`, `multer`) parses the request.
3. Handler responds with JSON echo of `body` and/or uploaded file metadata.

### Flow: Dev deploy (CI)

1. Push/merge to `dev` triggers Bitbucket Pipeline (**INFERRED** from file).
2. Pipeline SSHs to configured host and runs `git pull origin dev` in `$PATH_FOLDER`.

## 11. Things Other Repos May Depend On

- **Companion repo integration:** A **different repository** (**INFERRED** from team context) likely depends on this host for **reading project data/files** and **sending** updates (exact HTTP contract—REST vs raw static GET/PUT patterns—**NEEDS CONFIRMATION**; `index.js` only documents static `GET` for files and `POST` test routes, not a full project CRUD API).
- **Stable URL paths** for files under the repository root served by `express.static('.')` (exact paths **INFERRED** from product conventions in `AGENTS.md`: `app/<id>/…`, `plugin-settings/…`, `user/…`).
- **Directory and ID naming** as external path keys (do not rename `app/<id>` folders per `AGENTS.md`).
- **Legacy vs source schema** markers under `app/<id>/` (`app.json`, `data.db`, `src/`, etc.) for any tooling that packages or reads artifacts—**INFERRED** from `AGENTS.md`, not from runtime code in `index.js`.

## 12. Unknowns / Needs Confirmation

- **NEEDS CONFIRMATION:** Production authentication, TLS termination, and network isolation for a service that serves the full repo root and exposes `/print-headers` and `/test/cookies`.
- **NEEDS CONFIRMATION:** Complete list of internal clients and URL patterns they call.
- **NEEDS CONFIRMATION:** Whether `firebase-admin` will be enabled and which credentials path/env would be used (no template in repo).
- **NEEDS CONFIRMATION:** Whether `README.md` port `3999` or `index.js` port `3005` is authoritative for deployments (README is explicitly non-authoritative per `AGENTS.md`; **`index.js` is 3005**).
- **NEEDS CONFIRMATION:** Contents and tracking of `app/`, `tickets/`, and nested `AGENTS.md` files—ignored or partially ignored by git in this repo.
