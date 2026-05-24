# Service Map

## Overview

This map summarizes how **profiled eMOBIQ / Orangekloud repositories** connect: ownership, coarse interfaces, outbound dependencies, and participation in documented cross-repo flows. It is derived only from `repo-hub/repo-profiles/*.md` and `repo-hub/flows/*.md`. Endpoint specifics stay in repo profiles; table/model detail in `maps/database-map.md` when that map exists.

## Services

| Repo / Service | Purpose | Owns | Exposes | Calls / Depends On | Main Flows |
|---|---|---|---|---|---|
| `autopilot-debug-agent` | AI debugger and IDE-style assistant (Remix) embedded in the main web app; chat, previews, checkpoints, Autopilot-backed artifacts and billing. | Debugger UX and server-side LLM/chat pipeline; no first-party relational DB in-repo. | Summarized: Remix UI routes (`/`, `/chat/:id`, `/preview/...`); `/api/*` for chat, models, checkpoints, codes transform, logging, Autopilot import. | `emobiq-main-backend` (OAuth token introspect); **Autopilot** HTTP API (projects, documents, chats, pricing, runs, checkpoints, integration/plugins/codes); **Service Manager** (`editor-data`); LLM vendor APIs; optional MCP (e.g. Supabase). **Outbound HTTP may egress via `emobiq-v5-platform` `/connector/raw/`.** | Iframe bootstrap / SSO with parent; chat + checkpoints + billing; embedded preview; log housekeeping sidecar. |
| `emobiq-v5-platform` | Legacy multi-surface platform: visual editor, Cordova runtime, JSON API, connectors, mobile builder tooling, Node WebSocket helper. | Platform metadata and per-app data (MariaDB + per-app SQLite); connector/proxy behavior including **`/connector/raw/`**. | Summarized: `api/?controller=&action=` REST; **`/connector/raw/`** (and v2); `/edit/`, `/www3/`; econnector (Socket.IO, separate port). | **Compiler** (`POST` compile); **socket** `broadcast`; SCP (inferred); connector downstream hosts; **inbound proxy for `autopilot-debug-agent`**. | Editor, compile API, runtime preview; HTTP proxy for debug-agent external calls. |
| `emobiq-storage` | Node/Express static host for stored project payloads and test/diagnostic routes. | Filesystem layout under service root (`app/`, paths per product conventions; much gitignored). | Summarized: `express.static` GET for arbitrary paths; `/test/*` and header echo routes. **Auth: none in sample code** (deployment NEEDS CONFIRMATION). | No outbound HTTP in scanned application code. | Serve static artifacts; diagnostic POSTs; CI deploy pull. |
| `emobiq-socket` | Go WebSocket hub plus HTTP broadcast API; token or API-key auth. | In-memory channels/clients at runtime; Mongo wired for migrations / optional paths **not initialized from `main.go`**. | `GET /v1/ws/:channelName`, `POST /v1/broadcast`. | `emobiq-main-backend` (`POST` OAuth token introspect). | Authenticated subscribe; server-side broadcast; token validation. |
| `emobiq-main-frontend` | React SPA for authenticated platform use: projects, AI workspace, marketplace, billing UX, debug iframe. | Browser-only state; no backend DB. | None (static assets). | `emobiq-main-backend` (CRUD, OAuth, cookies proxy, application-control); **client backend**; **AI backend**; **subscription** service URLs; **service manager**; **storage** URL asset fetches; **WebSocket** (`REACT_APP_SOCKET_URL`); **`autopilot-debug-agent`** iframe + `postMessage`; SSO navigation; connector OAuth callback base (inferred). | OAuth login; AI project workspace + debug iframe; marketplace/applications; subscription links/APIs; Supabase OAuth popup. |
| `emobiq-main-backend` | Central Go API: users/companies, OAuth, applications, builds, publish, integrations, support, autopilot-related routes, rate limiting. | MariaDB platform data; per-application SQLite files; optional **Redis** (connector raw rate limits). | Summarized: `/v1/*` REST (oauth, applications, application-control, cookies + **v5-proxy-request**, audits, support, external APIs, autopilot, plugins, supabase proxy, **rateLimit**, SSO sync, etc.). | MariaDB, SQLite, optional Redis; SSO; **license / subscription** API; **compiler**, **publisher**, **parser**, **AI backend**; **service manager**; Supabase; SMTP; Google/Apple publish APIs; arbitrary URLs via cookie proxy (validated). | OAuth; application build → compiler handoff; publish; SSO company/user sync; Supabase proxy; scheduled cleanups. |
| `emobiq-compiler` | Go compile server: expands tarballs, runs Cordova/RN/Java pipelines, AI Cordova path, hands off to builder. | Temp workspace, locks, templates under `backend/storage`. | `POST /v1/compile`, `POST /v1/ai-compile` (API-key + platform header). | **Caller-supplied platform URL** (`emobiq_url`) for logger/build complete/clear; **socket** `/broadcast`; **builder** `POST /v1/build/application`; **service manager** package GET; SCP; config-driven URLs for AI/RN paths. | Standard compile; AI Cordova compile; migration CLI (operator). |
| `emobiq-builder` | Go native build orchestration after compiler handoff. | Staged tarballs, build locks, toolchain config. | `POST /v1/build/application` (API-key + `API-Key-Platform`). | **Caller-supplied** `emobiq_url` (logs, build complete, clear); **socket** `/broadcast`; SCP upload targets; local toolchains. | Async build; startup stale-lock recovery. |
| `autopilot-ai` | AI backend for generating and iterating AI projects (runs, documents, assets). | **NEEDS CONFIRMATION** — not profiled in `repo-hub`; persistence described only in flow doc. | **NEEDS CONFIRMATION** — flow cites `api/v1/...` patterns. | **NEEDS CONFIRMATION** | AI project generation/editing (per `flows/ai-projects-compiling-flow.md`). |
| `orangekloud-subscription` | Subscription checkout and plan/attribute storage; creates connector API keys via main-backend. | **NEEDS CONFIRMATION** — `orders`, `subscriptions`, `subscription_attributes` per flow doc only. | **NEEDS CONFIRMATION** — e.g. `POST /api/open/get_subscription_attr_by_idp` per flow. | `emobiq-main-backend` (`POST /v1/sso/user/:external_sso_user_id/api-keys`). | Post-checkout API key creation with limits (per `flows/create-send-request-api-keys.md`). |
| `emobiq-service-manager` | Plugin/package metadata for Cordova/RN compile (referenced name in flow). | **NEEDS CONFIRMATION** — no `repo-hub` repo profile. | **NEEDS CONFIRMATION** — e.g. `GET /package?id=...` per flow/compiler profile. | **NEEDS CONFIRMATION** | AI Projects compile — plugin resolution (per `flows/ai-projects-compiling-flow.md`). |

## Service Relationships

| From | To | Relationship | Evidence / Source | Confidence |
|---|---|---|---|---|
| `emobiq-main-frontend` | `emobiq-main-backend` | Browser HTTP client for core APIs, OAuth, application-control, cookies proxy | `repo-profiles/emobiq-main-frontend.md` | High |
| `emobiq-main-frontend` | `autopilot-debug-agent` | Embeds debugger iframe; `postMessage` auth and lifecycle | `repo-profiles/emobiq-main-frontend.md`, `repo-profiles/autopilot-debug-agent.md` | High |
| `emobiq-main-frontend` | `emobiq-socket` | WebSocket client for realtime channels | `repo-profiles/emobiq-main-frontend.md` | High |
| `emobiq-main-frontend` | **Subscription / AI / client / service-manager / storage** | HTTP or URL construction per env | `repo-profiles/emobiq-main-frontend.md` | High |
| `autopilot-debug-agent` | `emobiq-main-backend` | Validates access token via introspection | `repo-profiles/autopilot-debug-agent.md` | High |
| `autopilot-debug-agent` | **Autopilot backend** | REST for projects, documents, chats, billing, checkpoints, codegen | `repo-profiles/autopilot-debug-agent.md` | High |
| `autopilot-debug-agent` | **Service Manager** | Plugin/editor data HTTP | `repo-profiles/autopilot-debug-agent.md` | High |
| `autopilot-debug-agent` | `emobiq-v5-platform` | Outbound HTTP proxied through **`/connector/raw/`** | `repo-profiles/emobiq-v5-platform.md` (documented ops use; **INFERRED** end-to-end wiring in agent repo—agent not named in v5 source tree) | Medium |
| `emobiq-v5-platform` | `emobiq-compiler` | Submit compile job HTTP | `repo-profiles/emobiq-v5-platform.md` | High |
| `emobiq-v5-platform` | `emobiq-socket` | POST broadcast for build events | `repo-profiles/emobiq-v5-platform.md` | High |
| `emobiq-v5-platform` | `emobiq-main-backend` | Raw connector v2 gateway calls **`POST /v1/rate-limit`** before delegating to legacy Data controller | `flows/create-send-request-api-keys.md`; `repo-profiles/emobiq-main-backend.md` | High |
| `emobiq-v5-platform` | `emobiq-main-backend` | **NEEDS CONFIRMATION:** legacy `mainBackendPath` / platform URLs in samples | `repo-profiles/emobiq-v5-platform.md` | Low |
| `emobiq-main-backend` | `emobiq-compiler` | POST compile / ai-compile; optional SCP tarball | `repo-profiles/emobiq-main-backend.md`, `repo-profiles/emobiq-compiler.md` | High |
| `emobiq-main-backend` | **Redis** | Connector raw V2 rate limits (optional) | `repo-profiles/emobiq-main-backend.md` | High |
| `emobiq-compiler` | `emobiq-builder` | POST `/v1/build/application` after SCP handoff | `repo-profiles/emobiq-compiler.md`, `repo-profiles/emobiq-builder.md` | High |
| `emobiq-compiler` | `emobiq-socket` | POST `/broadcast` for compile status | `repo-profiles/emobiq-compiler.md` | High |
| `emobiq-compiler` | **Platform (`emobiq_url`)** | Logger, build complete, clear — **target is caller-supplied** (may be v5 API patterns or other deployment) | `repo-profiles/emobiq-compiler.md` | Medium |
| `emobiq-compiler` | **Service Manager** | GET package metadata | `repo-profiles/emobiq-compiler.md` | High |
| `emobiq-builder` | **Platform (`emobiq_url`)** | Logs, build complete, clear | `repo-profiles/emobiq-builder.md` | High |
| `emobiq-builder` | `emobiq-socket` | POST `/broadcast` | `repo-profiles/emobiq-builder.md` | High |
| `emobiq-socket` | `emobiq-main-backend` | OAuth token introspection | `repo-profiles/emobiq-socket.md` | High |
| `orangekloud-subscription` | `emobiq-main-backend` | Create/update user API keys after checkout | `flows/create-send-request-api-keys.md` | High |

## Important Cross-Repo Flows

| Flow | Repos Involved | Summary | Detailed Doc |
|---|---|---|---|
| AI projects compiling | `emobiq-main-frontend`, `emobiq-main-backend`, `emobiq-compiler`, `emobiq-builder`, `emobiq-socket`, `emobiq-storage`, `autopilot-ai`, `emobiq-service-manager` (+ `autopilot-debug-agent` for debug UI, not compile orchestrator) | UI triggers build on main-backend; main-backend invokes compiler (incl. optional **`/v1/ai-compile`** for AI Cordova); compiler hands off to builder; compiler/builder callback to platform URLs and push socket **`/broadcast`**; storage hosts static AI asset URLs when wired. | `repo-hub/flows/ai-projects-compiling-flow.md` |
| Create API keys + connector rate limit | `orangekloud-subscription`, `emobiq-main-backend`, `emobiq-v5-platform` | Checkout creates connector API key + limits in main-backend; raw V2 gateway calls **`POST /v1/rate-limit`** before legacy Data proxy. | `repo-hub/flows/create-send-request-api-keys.md` |
| Login (OAuth / SSO) | `emobiq-main-frontend`, `emobiq-main-backend`, `orangekloud-sso`, `orangekloud-subscription` | SPA login via main-backend **`/v1/oauth/login`** → IdP → **`/v1/callback`**; subscription **`/sso/login`** → **`/callback`**; SSO Passport **`/oauth/authorize`**, **`/oauth/token`**; optional **`/login/authorize?token=`** and **`oauth_chain`** bootstrap; **`_em_id`** cookie alignment. | `repo-hub/flows/login-flow.md` |

## Known Gaps / Needs Confirmation

- **`autopilot-debug-agent` → `emobiq-v5-platform`:** Connection is **deployment/ops context** in the v5 profile (agent name not in v5 source); exact URL, auth, and contract **NEEDS CONFIRMATION** in agent config.
- **`emobiq-main-backend` vs `emobiq-v5-platform` HTTP routing:** When compiler/builder `emobiq_url` targets **v1 query-string API** vs **`/v1` Go API** is **deployment-specific**; flow doc notes ambiguity for **`/v1/ai-compile`** vs **`/v1/compile`** selection for AI apps.
- **`emobiq-main-frontend` ↔ `emobiq-socket` path:** Frontend uses `{SOCKET_URL}/ws/{channel}`; socket service exposes `/v1/ws/:channelName` — **gateway/path NEEDS CONFIRMATION** (noted in `flows/ai-projects-compiling-flow.md`).
- **`emobiq-storage`:** Production auth/TLS and full client list **NEEDS CONFIRMATION**; companion repo consumption is **INFERRED** in storage profile.
- **`autopilot-ai`, `orangekloud-subscription`, `emobiq-service-manager`:** No `repo-hub/repo-profiles/*.md` entries; rows above use **only** flow documents — details **NEEDS CONFIRMATION** until profiles exist.
- **`emobiq-v5-platform`:** Apache public URL mapping, auth coverage on several API actions, and full JS outbound inventory are **NEEDS CONFIRMATION** per profile.
