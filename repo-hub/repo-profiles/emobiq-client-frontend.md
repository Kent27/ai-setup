# Repo Profile: emobiq-client-frontend

Last updated: 2026-05-07
Confidence: High (core stack, routing, auth, env template); Medium (exhaustive list of every API path variant; product-line ownership)

## 1. Purpose

Visual app builder **client**: SPA for editing applications (pages/snippets, event flows, global config, services, database bindings, localization, plugins, themes, media, build/publish). Talks to separate **client** and **main** backends plus connector, storage, preview, websocket, AI, and (optionally) AWS APIs from the browser.

### Product / program context

**Emobiq AI** does not use this repository in any substantial way. This codebase belongs to the **previous Emobiq No Code** product line (visual no-code builder), not to the Emobiq AI application surface.

The **only** integration here that reaches toward the Emobiq AI stack is **`src/common/helper/Parser.js`**: it calls `${REACT_APP_AI_API_URL}` (for example `/database/retrieve_all_files/...` and `/azure_storage/get_blob/...`) to pull generated HTML and turn it into editor pages. That flow is **legacy relative to Emobiq AI** and has **not been exercised or regression-tested** for a long time (per team); treat it as **unmaintained / high risk** if re-enabled.

## 2. Tech Stack

| Area | Detail |
|---|---|
| Language | JavaScript (+ some TypeScript) |
| Framework | React 17 |
| Routing | react-router-dom v5 |
| State | Redux + Immer (custom API middleware) |
| Build | Create React App (`react-scripts` 5.x) |
| Package manager | npm |
| Database | No server-side DB in this repo; data via backend HTTP APIs |
| Other notable libs | axios, jQuery/Bootstrap UI, Primereact, Fine Uploader, i18next, aws-sdk (DynamoDB helper) |

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `src/App.js`, `src/index.js` | Application shell and React mount |
| `src/Route.js` | Browser routes and protected editor entry |
| `src/common/route/Protected.js` | Auth gate + session bootstrap |
| `src/common/hooks/SessionRefresh.js` | Token introspection on load |
| `src/common/helper/Redirect.js` | SSO redirect/popup to main frontend login |
| `src/common/redux/middleware/Api.js` | Central Redux `api` actions: axios, Bearer cookie, 401→login |
| `src/common/helper/HttpRequest.js` | Non–Redux-path HTTP (axios) including introspect |
| `src/common/data/Constant.js` | Cookie key name for token (`API_TOKEN`), compiler constants |
| `src/page/design/` | Main visual editor |
| `src/library/design/` | Design-time components, properties, drag/drop |
| `src/common/redux/` | Actions, handlers, store |
| `public/locales/` | Static locale JSON (i18next-http-backend) |
| `.env.sample` | Canonical list of `REACT_APP_*` integration URLs and flags |
| `README.md` | Setup (Node/npm, copy `.env`) and directory overview |
| `src/common/helper/Parser.js` | Legacy **Emobiq AI HTML → pages** import path (`REACT_APP_AI_API_URL`); see §1 product context |

## 4. Exposed Endpoints

This repo does not appear to expose HTTP API endpoints. It ships a **static SPA** (dev: CRA dev server; prod: static host). “Routes” below are **client-side** React Router paths.

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| N/A (client) | `/auth` | OAuth callback: read `token`/`state` from query, set cookie, redirect or close popup | `src/page/auth/index.js` | Receives token from main app flow |
| N/A (client) | `/editor`, `/editor/*` | Protected editor surfaces (design, global, service, language, modules, help, database, build-publish) | `src/Route.js`, `src/common/route/Protected.js` | Yes (session + introspect) |
| N/A (client) | `/not-authorized`, `/not-found` | Error/exit pages | `src/Route.js` | No |

## 5. Outbound API Calls

Traffic is mostly **HTTPS to configured bases** (`process.env.REACT_APP_*`), plus **WebSocket**, **connector** URLs, **V5 preview** iframe `src`, a **legacy Emobiq AI HTML import** via `Parser.js`, and **browser AWS SDK** for DynamoDB screens.

| Target Service / Host | Method | Endpoint / URL pattern | Purpose | Source file(s) |
|---|---|---|---|---|
| Client backend | varies | Paths under `${REACT_APP_CLIENT_BACKEND_URL}` e.g. `/app`, `/pages`, `/media`, `/database/tables`, `/database/data`, `/applications`, `/plugins`, `/style`, `/packages`, `/logs/:appId` | Core editor data CRUD | `src/common/redux/action/*.js`, `.env.sample`, `src/common/component/MediaLibrary.js`, `src/page/build-publish/component/LogHistoryView.js` |
| Main backend | varies | e.g. `/applications/...`, `/oauth/tokens/introspect`, `/help`, `/tools/code-parser/import|export`, `/tools/automation/page/execute`, application-control URLs from env | Session, builds/publish, import/export, automation, licensing-style controls | `src/common/hooks/SessionRefresh.js`, `src/common/redux/action/PublishApp.js`, `src/page/design/component/modal/CodeParser.js`, `src/page/design/component/modal/PageAutomation.js`, `src/common/helper/ImplementControl.js`, `src/common/component/modal/Help.js`, `src/common/helper/Parser.js` |
| Connector | GET/POST | `${REACT_APP_CONNECTOR_URL}` + `${REACT_APP_CONNECTOR_NAV_PATH}`, `..._ACUMATICA_PATH`, `..._SAGE_PATH` with query APIs | NAV/Acumatica/SAGE wizard/API discovery | `src/page/design/component/wizard/Helper.js` |
| Connector V2 | POST | `${REACT_APP_CONNECTOR_V2_URL}/dynamics365bc/*` | Dynamics 365 BC verify/service/action lists | `src/library/design/property/Elements.js`, `src/page/design/component/action/ParamDropdown.js` |
| Storage / CDN | GET | `${REACT_APP_STORAGE_URL}/...` | Default images, app asset paths | `src/common/helper/Path.js`, `src/common/helper/Theme.js`, `src/library/design/component/Image.js` |
| Emobiq AI–related HTTP (legacy) | GET | `${REACT_APP_AI_API_URL}` e.g. `/database/retrieve_all_files/...`, `/azure_storage/get_blob/...` | Fetch generated HTML for import into the no-code editor | `src/common/helper/Parser.js` — **stale / rarely tested** (see §1) |
| Push notifications | varies | `${REACT_APP_PUSH_NOTIFICATION_URL}` | Non-visual config surface | `src/common/data/NonVisual.js` |
| WebSocket | WS | `${REACT_APP_SOCKET_URL}/ws/` + `${REACT_APP_SOCKET_BROADCAST_CHANNEL}` | Live build/publish status | `src/common/component/SocketStatus.js` |
| V5 preview (iframe) | GET | `${REACT_APP_V5_PLATFORM_PREVIEW_URL}/www3/...` | In-app preview | `src/common/component/modal/Modal.js` (`DesignPreview`), `src/common/component/general/Header.js` |
| AWS DynamoDB (public AWS API) | SDK | DynamoDB operations with **user-supplied** keys in UI | Optional database editor flows | `src/common/helper/DynamoDB.js`, `src/page/database/component/AWSDynamoDB.js` |

Additional ad hoc `fetch` calls exist for uploads and connector verification (see `Tools.js`, `Elements.js`, `ParamDropdown.js`).

## 6. Database / Models / Tables

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| _(none locally)_ | N/A — no migrations/ORM in repo | — | — |
| App database tables / rows (logical) | User-defined tables and data edited in UI | Via client backend REST (`REACT_APP_TABLE_URL`, `REACT_APP_TABLE_DATA_URL`) | `src/common/redux/action/FetchTableListData.js`, `SetNewTableData.js`, etc. |
| DynamoDB tables (logical) | Optional AWS DynamoDB management from Database section | Via **browser** aws-sdk (`DynamoDB.js`) **INFERRED**: calls AWS DynamoDB endpoints | `src/page/database/component/AWSDynamoDB.js` |

## 7. Jobs / Queues / Cron / Workers

No jobs, queues, cron tasks, or workers were found.

Realtime updates use a **browser WebSocket** client (`SocketStatus.js`), not a background worker process.

## 8. Configuration & Environment

### Primary: CRA `process.env.REACT_APP_*` (via `.env` at build/runtime)

Built with Create React App: integration URLs and flags are **`REACT_APP_` prefixed** and baked at build time. Local setup copies **`.env.sample` → `.env`** (see `README.md`).

| Key / Name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `REACT_APP_CLIENT_BACKEND_URL` | Base URL for client API (`/v1`) | `.env.sample`, many `src/common/redux/action/*.js` |
| `REACT_APP_CLIENT_FRONTEND_URL` | This app origin (auth redirect targets) | `.env.sample`, `src/common/helper/Redirect.js` |
| `REACT_APP_CLIENT_API_KEY` | **Cookie name** for bearer token storage (default `_em_cid` if unset) | `.env.sample`, `src/common/data/Constant.js` |
| `REACT_APP_MAIN_BACKEND_URL` | Main platform API + OAuth introspect | `.env.sample`, `SessionRefresh.js`, publish/build actions |
| `REACT_APP_MAIN_FRONTEND_URL` | Main app (login authorize, nav links) | `.env.sample`, `Redirect.js`, `Header.js`, error pages |
| `REACT_APP_*_URL` variants | Per-domain paths: `APPLICATION_URL`, `PAGE_URL`, `MEDIA_URL`, `GLOBAL_URL`, `THEME_URL`, connectors, storage, socket, preview, docs, automation, parsers, AI, push | `.env.sample` — consumers vary; grep `REACT_APP_` under `src/` |
| `REACT_APP_SOCKET_URL`, `REACT_APP_SOCKET_BROADCAST_CHANNEL` | Websocket endpoint + channel | `.env.sample`, `SocketStatus.js` |
| `REACT_APP_API_QUERY_LIMIT`, `REACT_APP_API_PAGE_SIZE` | Listing pagination defaults | `.env.sample`, `src/common/helper/Helper.js` |
| `REACT_APP_DESIGN_PREVIEW_CACHE_BUSTER` | Cache bust query for preview assets | `.env.sample`, `src/common/helper/Style.js` |
| `PORT`, `GENERATE_SOURCEMAP`, `REACT_APP_DISABLE_STRICT_MODE` | Dev/build behavior | `.env.sample`, CRA |

### Other / Secondary

| Variable / Key | Purpose | Used In |
|---|---|---|
| `NODE_ENV` | Standard CRA/React env | CRA toolchain |
| Locale files | Served as static `/locales/{{lng}}.json` | `public/locales/`, `src/common/localization/i18n.ts` |

**Note:** `.env.sample` declares some URLs (e.g. `REACT_APP_PAGE_URL`, `REACT_APP_SNIPPET_URL`, `REACT_APP_USER_URL`, `REACT_APP_NOTIFICATIONS_URL`) that **do not** appear referenced under `src/` in this scan—they may be legacy or consumed indirectly; see §12.

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| Client backend | REST API | Primary editor/app data | `middleware/Api.js`, `REACT_APP_*` URL actions |
| Main backend | REST API + OAuth | Login/session introspect, builds, publish, tools, help | `SessionRefresh.js`, publish/build actions |
| Main frontend | Auth / SSO | `/login/authorize` flow and return navigation | `Redirect.js` |
| Connector / Connector V2 | REST | ERP/BC integrations in wizards/properties | `wizard/Helper.js`, `Elements.js` |
| Object storage / CDN | HTTP | Themes, icons, media URLs | `Path.js`, `Theme.js`, uploads |
| V5 preview host | HTTP (iframe) | Device preview | `Modal.js`, `Header.js` |
| Websocket gateway | WS | Push status for builds/publish | `SocketStatus.js` |
| Emobiq AI (legacy parser path) | REST | Optional HTML retrieval + client-side parse into pages; **not** part of current Emobiq AI UX | `Parser.js` (see §1: unmaintained) |
| Push notification host | REST | Optional feature wiring | `NonVisual.js` |
| AWS DynamoDB | HTTPS (AWS SDK) | Optional tables UI | `DynamoDB.js` |

## 10. Main Flows

### Flow: SSO login and protected editor

1. User hits protected route; Redux `user.valid` may be unset → loader.
2. `SessionRefresh` POSTs Bearer token from cookie to `{MAIN_BACKEND}/oauth/tokens/introspect`.
3. On success, user is stored in Redux; on failure, `popupLogin` opens main frontend authorize with `state` containing `returnUrl` and `redirectUrl` → client `/auth`.
4. `/auth` reads `token` from query, writes cookie named by `REACT_APP_CLIENT_API_KEY` (or `_em_cid`), redirects to `returnUrl` or closes popup.

### Flow: Visual edit and save

1. Editor loads app via client backend (`REACT_APP_APPLICATION_URL` and related actions).
2. `#editor-view` iframe (`react-frame-component`) renders page tree; scripts/plugins may access `window.frames['editor-view']`.
3. Changes dispatch Redux handlers; persists via `type: 'api'` middleware to client/main backends as applicable.

### Flow: Build / publish awareness

1. User triggers publish/build through main-backend actions (`PublishApp`, `FetchBuilds`, etc.).
2. `SocketStatus` subscribes to `${REACT_APP_SOCKET_URL}/ws/` and may reconcile with `{MAIN_BACKEND}/applications/.../build|publish`.

### Flow: DynamoDB helper (database section)

1. User enters AWS keys in UI; `DynamoDB` class configures `aws-sdk` in browser (see `AWSDynamoDB.js`).

### Flow: Legacy AI-generated HTML import (No Code only)

1. Code calls `fetchGeneratedHtml` / related helpers in `Parser.js` with a project key.
2. Responses are downloaded from `${REACT_APP_AI_API_URL}` and parsed into pages/styles in the editor.
3. **Operational note:** This path is tied to the old no-code product; it is not validated for current Emobiq AI deployments (see §1).

## 11. Things Other Repos Depend On

- **SSO redirect contract**: main frontend must implement `/login/authorize?state=` where `state` JSON includes `returnUrl`, `redirectUrl` pointing to this app’s **`/auth`**, optionally `isPopUp` (`src/page/auth/index.js`, `src/common/helper/Redirect.js`).
- **Token cookie name**: configurable via `REACT_APP_CLIENT_API_KEY` cookie key (`src/common/data/Constant.js`); backends must issue tokens readable by this name.
- **OAuth introspect**: main backend **`POST ${REACT_APP_MAIN_BACKEND_URL}/oauth/tokens/introspect`** with `Authorization: Bearer <token>` expected by session bootstrap (`SessionRefresh.js`).
- **Stable client routes**: `/editor` and subpaths in `src/Route.js` are deep-link and returnUrl targets.
- **Editor iframe contract**: DOM id/name **`editor-view`** and `window.frames['editor-view']` access for plugin/runtime scripts (`src/library/design/component/Plugin.js`, `AGENTS.md` guidance).
- **Preview URL shape**: consumers building preview iframes should match `${REACT_APP_V5_PLATFORM_PREVIEW_URL}/www3/{appId}&...` pattern used in `DesignPreview` (`src/common/component/modal/Modal.js`).
- **API error behavior**: 401 triggers login popup; some labels redirect to `/not-found` (`Api.js`).

## 12. Unknowns / Needs Confirmation

- NEEDS CONFIRMATION: Whether `REACT_APP_PAGE_URL`, `REACT_APP_SNIPPET_URL`, `REACT_APP_USER_URL`, and `REACT_APP_NOTIFICATIONaS_URL` in `.env.sample` are still required by any deployment or removed from active use (no `src/` references found in this scan).
- NEEDS CONFIRMATION: Whether `${REACT_APP_AI_API_URL}` endpoints used by `Parser.js` still match a live Emobiq AI (or successor) deployment; **compatibility is unverified** after long period without testing.
- INFERRED: Exact production hostnames and network policies for each backend are environment-specific, not in repo.
