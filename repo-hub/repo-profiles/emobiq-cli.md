# Repo Profile: emobiq-cli

Last updated: 2026-05-07
Confidence: Medium

## 1. Purpose

CLI for managing eMOBIQ plugin projects, including login, initialization, build/test, and publishing against the eMOBIQ service manager.

## 2. Tech Stack

- Language: JavaScript (Node.js CommonJS)
- Framework: None (CLI)
- Runtime: Node.js
- Package manager: npm
- Database: None
- Other important dependencies: `request`, `rollup`, `colors`, `tough-cookie`, `jsdom`

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `bin/emobiq` | CLI entrypoint and command dispatch |
| `lib/cmd.js` | Command registry and auth middleware wiring |
| `lib/cmd/` | Command handlers (`login`, `init`, `build`, `publish`, etc.) |
| `lib/func.js` | Shared helpers, API client, packaging, and plugin pull/install |
| `lib/auth/` | SSO login, token storage, auth helpers |
| `config.js` | Local runtime configuration (service URLs, SSO settings) |
| `config.js.sample` | Template for configuration values |
| `README.md` | Basic usage and overview |

## 4. Exposed Endpoints

This repo does not appear to expose HTTP endpoints.

Local-only note: the SSO login flow spins up a temporary localhost callback server (`GET /callback`) during `emobiq login`. (`lib/auth/sso.js`)

## 5. Outbound API Calls

| Target Service / Host | Method | Endpoint / URL | Purpose | Source file |
|---|---|---|---|---|
| eMOBIQ Service Manager (`serviceManagerURL`) | POST | `/api/cli-login` | Exchange SSO token for CLI session | `lib/cmd/login.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | GET | `/api/plugin/access` | Check plugin creation permission | `lib/cmd/login.js`, `lib/middleware/auth.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | POST | `/api/sso-logout` | SSO logout | `lib/cmd/logout.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | GET | `/api/logout` | CLI logout | `lib/cmd/logout.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | GET | `/api/me` | Auth validation / current user | `lib/func.js`, `lib/cmd/whoami.js`, `lib/middleware/auth.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | GET | `/api/package/:id` | Check plugin availability/metadata | `lib/cmd/init.js`, `lib/cmd/publish.js`, `lib/func.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | POST | `/api/package/` | Create new plugin package | `lib/cmd/publish.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | PUT | `/api/package/:id` | Update existing plugin package | `lib/cmd/publish.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | GET | `/api/deps/?id=...` | Fetch dependency list | `lib/func.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | GET | `/api/archive/package/:id/:version.tar` | Download plugin package archive | `lib/func.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | GET | `/v1/cli/token?sessionId=...` | Exchange SSO session for token | `lib/auth/sso.js` |
| eMOBIQ Service Manager (`serviceManagerURL`) | GET (redirect target) | `/v1/callback-cli` | OAuth redirect for SSO | `lib/auth/sso.js` |
| SSO server (`ssoUrl`) | GET | `<ssoUrl><ssoAuthPath>` | OAuth authorization endpoint | `lib/auth/sso.js` |
| eMOBIQ app assets (`emobiqApp.url`) | GET | `script/...` assets | Build/test preview resources | `config.js.sample`, `lib/cmd/test.js` |
| Google Maps | GET | `http://maps.googleapis.com/maps/api/js` | Preview resource | `config.js.sample` |

## 6. Database / Models / Tables

No database usage found in this CLI repo.

## 7. Jobs / Queues / Cron / Workers

No jobs, queues, cron tasks, or workers were found.

## 8. Configuration & Environment

### Primary: `config.js`

Local configuration module imported by CLI runtime. A committed template exists at `config.js.sample`.

| Key / Name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `serviceManagerURL` | Base URL for eMOBIQ Service Manager APIs | `config.js.sample`, `lib/func.js`, `lib/cmd/login.js` |
| `emobiq` | Base URL for eMOBIQ web assets | `config.js.sample`, `lib/cmd/test.js` |
| `ssoUrl` | SSO server base URL | `config.js.sample`, `lib/auth/sso.js` |
| `ssoClientId` | SSO OAuth client ID | `config.js.sample`, `lib/auth/sso.js` |
| `ssoAuthPath` | OAuth authorization path | `config.js.sample`, `lib/auth/sso.js` |
| `emobiqApp.res` | Asset URLs used for preview tests | `config.js.sample`, `lib/cmd/test.js` |

### Other / Secondary

| Variable / Key | Purpose | Used In |
|---|---|---|
| `~/.emobiq-cli.json` | Token + session cookie storage | `lib/auth/tokenStorage.js`, `lib/func.js` |
| CLI args (`process.argv`) | Publish flags (`--public` / `--private`) | `lib/cmd/publish.js` |

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| eMOBIQ Service Manager (`serviceManagerURL`) | API | Auth, package metadata, upload, download | `lib/func.js`, `lib/cmd/publish.js`, `lib/cmd/login.js` |
| eMOBIQ SSO (`ssoUrl`) | Auth | OAuth login and token exchange | `lib/auth/sso.js`, `config.js.sample` |
| eMOBIQ app assets (`emobiqApp.url`) | Web assets | Preview/test runtime resources | `config.js.sample`, `lib/cmd/test.js` |
| Google Maps JS | Web asset | Preview/test dependency | `config.js.sample` |

## 10. Main Flows

### Flow: SSO login and CLI session setup

1. Start local callback server and open OAuth URL.
2. Exchange `sessionId` for token via `/v1/cli/token`.
3. Call `/api/cli-login` to establish CLI session cookies.
4. Optionally verify plugin access with `/api/plugin/access`.

### Flow: Plugin init/build/test/publish

1. `plugin init` checks `/api/package/:id`, scaffolds `emobiq.json` and platform folders.
2. `plugin build` generates `dist/` artifacts and updates `emobiq.json`.
3. `plugin test` runs build then preview tests (jsdom scaffolding).
4. `plugin publish` packs the project and uploads to `/api/package` (create/update).

### Flow: Plugin pull/install

1. Resolve package metadata via `/api/package/:id`.
2. Download archive from `/api/archive/package/:id/:version.tar`.
3. Extract into `esm_modules` or working directory.

## 11. Things Other Repos Depend On

- None identified; this repo is a CLI client that consumes eMOBIQ service APIs.

## 12. Unknowns / Needs Confirmation

- NEEDS CONFIRMATION: full response schemas and required fields for `/api/package` publish payloads.
