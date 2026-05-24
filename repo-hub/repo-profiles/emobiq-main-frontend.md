# Repo Profile: emobiq-main-frontend

Last updated: 2026-05-05

Confidence: High

## 1. Purpose

React single-page application for the **eMobiq** platform: authenticated users manage projects and companies, marketplace, account settings, support, and (with subscription/guards) **AI** project creation, templates, plugins, API management, builds, releases, and debug/preview workflows. OAuth/SSO login is delegated to **main-backend**; this repo is the browser client only.

## 2. Tech Stack

| Item | Detail |
|---|---|
| Language | TypeScript (strict) |
| Framework | React 18 |
| Runtime | Browser (CRA dev server locally; static `build/` in production behind a host) |
| Package manager | npm (`frontend/package.json`) |
| Database | None in-repo (no local DB schema; data via backend APIs) |
| Other important dependencies | Zustand + Immer, axios, react-router-dom v6, Bootstrap 5 / SCSS, Formik + Yup, i18next, jose |

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `frontend/` | CRA application root; `npm` scripts and env live here |
| `frontend/src/index.tsx` | App bootstrap (StrictMode toggle via env) |
| `frontend/src/App.tsx` | Application shell wrapping router + providers |
| `frontend/src/Route.tsx` | Client-side routes and guard composition |
| `frontend/src/page/` | One folder per screen/feature route |
| `frontend/src/common/` | Shared UI, hooks, helpers, layouts, guards, assets |
| `frontend/src/common/zustand/` | Global store: interfaces, slices, `Store.ts` |
| `frontend/src/common/helper/HttpRequest.ts` | Axios instances and auth interceptors for backends |
| `frontend/src/common/helper/Redirect.ts` | OAuth login URL construction for main-backend |
| `frontend/src/common/helper/Request.ts` | Main-backend **v5 cookie proxy** (`/v1/cookies/v5-proxy-request`) for external API calls |
| `frontend/src/common/hooks/Socket.ts` | WebSocket client to `REACT_APP_SOCKET_URL` (`/ws/{channel}`) |
| `frontend/public/locales/` | i18n JSON (e.g. `en.json`) |
| `frontend/.env.sample` | Canonical template for `REACT_APP_*` variables |
| `AGENTS.md`, `frontend/AGENTS.md` | Human/agent conventions and index |

## 4. Exposed Endpoints

This repo does not expose HTTP APIs. It is a **client** built with Create React App; production serving is static assets (optionally + HTTPS in dev via `npm start`). There is **no** in-repo Express/Node route layer.

## 5. Outbound API Calls

Calls use **axios** (`frontend/src/common/helper/HttpRequest.ts`) with Bearer token from cookies (`Cookies.ts`), unless noted. Paths are relative to each instance’s `baseURL` unless a full URL is built.

| Target Service / Host | Method | Endpoint / URL (representative) | Purpose | Source file(s) |
|---|---|---|---|---|
| Main backend (`REACT_APP_MAIN_BACKEND_URL`) | `GET`/`POST`/… | `v1/users/*`, `v1/companies/*`, apps/settings/support/cookies routes used in slices | Core platform CRUD and session-related APIs | `frontend/src/common/zustand/slice/*.ts`, `CookieSlice.ts`, `SupportSlice.ts`, … |
| Main backend | `POST` | `/v1/application-control/*` (create/storage/download/compile/publish) | Implementation-control / quotas | `frontend/src/common/helper/ImplementControl.ts` |
| Main backend (v5 proxy) | Via FormData POST | `{MAIN_BACKEND_URL}/v1/cookies/v5-proxy-request` | Proxied outbound HTTP with cookie handling for “v5” style APIs | `frontend/src/common/helper/Request.ts` |
| Client backend (`REACT_APP_CLIENT_BACKEND_URL`) | Various | Paths used for AI **build** and related flows (see slices/components) | Build / client-backend features | Uses `clientBackendHttpRequest` / `clientBackendBuildHttpRequest` in `HttpRequest.ts` consumers |
| AI backend (`REACT_APP_AI_BACKEND_URL`) | Various | e.g. `api/v1/projects/{id}/server` (load balancer URL), AI document/other `api/v1/*` routes | AI project VM URL resolution, AI APIs | `HttpRequest.ts` (`getLoadBalancerUrl`), `AiSlice.ts`, … |
| Service manager (`REACT_APP_SERVICE_MANAGER_URL`) | Various | Package/plugin HTTP APIs | Plugin marketplace integration | Components/slices referencing `serviceManagerHttpRequest`; icon URLs like `/api/package/{key}/icon/logo.png` |
| Subscription (`REACT_APP_SUBSCRIPTION_URL`) | Various | Dashboard/credits URLs opened or called via axios | Billing/subscription | `NavbarMain.tsx`, `AiNavbar.tsx`, `UserSlice.ts`, `LoginAuthorizeSection.tsx`, … |
| Object storage (`REACT_APP_STORAGE_URL`) | GET (asset URLs) | Constructed `/app/*`, `/user/*`, build artifact paths | Downloads, uploads, attachments | `Helper.ts`, support panels, build components |
| WebSocket gateway (`REACT_APP_SOCKET_URL`) | WebSocket | `{SOCKET_URL}/ws/{channel}` | Real-time channel per `Socket.ts` | `frontend/src/common/hooks/Socket.ts` |
| AI load balancer VM (resolved) | HTTP/WS | Dynamic host from AI backend response | Commands to per-project AI environment | `getLoadBalancerUrl` + `aiLoadBalancerBackendHttpRequest` |
| Connector / OAuth (`REACT_APP_NEW_CONNECTOR_URL`) | INFERRED OAuth callback | `{NEW_CONNECTOR_URL}/v1/oauth/callback` | External API domain OAuth callback URL shown/configured | `AIExternalApiDomainAuth.tsx` |
| Debug agent (`REACT_APP_DEBUG_AGENT_URL`) | Embed + `postMessage` | `/preview/...`, `/chat/...` | AI project debug iframe and origin checks | `ProjectDesignPreview.tsx`, `Debug.tsx`, `Overview.tsx`, `ai-project/index.tsx` |
| SSO (`REACT_APP_SSO_URL`) | Navigate | Used on logout (_em_id cookie clearance) | Logout handshake | `modals/Logout.tsx` |

For a fuller path inventory, grep `httpRequest.` / `aiBackendHttpRequest.` / etc. under `frontend/src/common/zustand/slice/` and `frontend/src/page/`.

## 6. Database / Models / Tables

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| *(none persisted locally)* | Domain types mirror API payloads in TypeScript interfaces | N/A | Various `frontend/src/common/zustand/interface/` and page-level types |

## 7. Jobs / Queues / Cron / Workers

No jobs, queues, cron tasks, or workers were found. The app uses **`setInterval`** in a few UI flows for polling/popup checks only (not server-side scheduling).

## 8. Configuration & environment

### Primary: `.env` (Twelve-Factor, CRA)

Runtime integration settings are read from **`process.env.REACT_APP_*`** at **build time** via Create React App (values baked into the bundle). The committed template is `frontend/.env.sample`; developers copy to `frontend/.env`. No separate checked-in `config.ts` for service URLs—all are env-driven as consumed in TSX/TS.

| Key / name | Purpose (no secret values) | Evidence (sample + consumer files) |
|---|---|---|
| `REACT_APP_MAIN_BACKEND_URL` | Main API + OAuth base | `.env.sample`; `HttpRequest.ts`, `Redirect.ts`, `ImplementControl.ts`, `Request.ts` |
| `REACT_APP_MAIN_BACKEND_VERSION`, `REACT_APP_MAIN_BACKEND_LOGIN_PATH` | OAuth login path segments | `.env.sample`; `Redirect.ts` |
| `REACT_APP_MAIN_BACKEND_API_KEY` | Server-style `Api-Key` header for some calls | `.env.sample`; `NotificationSlice.ts`, `ApplicationSlice.ts` |
| `REACT_APP_SSO_URL` | Logout redirect to clear SSO/session | `.env.sample`; `modals/Logout.tsx` |
| `REACT_APP_CLIENT_BACKEND_URL` | Client / build backend | `.env.sample`; `HttpRequest.ts` |
| `REACT_APP_AI_BACKEND_URL` | AI services | `.env.sample`; `HttpRequest.ts`, `AiSlice.ts` |
| `REACT_APP_SERVICE_MANAGER_URL` | Service manager APIs + links | `.env.sample`; `HttpRequest.ts`, `NavbarMain.tsx`, `PluginMarketplace.tsx` |
| `REACT_APP_SUBSCRIPTION_URL` | Subscription/billing URLs | `.env.sample`; multiple components + `subscriptionHttpRequest` |
| `REACT_APP_SOCKET_URL` | WebSocket base | `.env.sample`; `Socket.ts` |
| `REACT_APP_STORAGE_URL` | File/asset base URLs | `.env.sample`; `Helper.ts`, support components |
| `REACT_APP_CLIENT_FRONTEND_URL`, `REACT_APP_SERVER_FRONTEND_URL` | Editor/preview routing to other frontends | `.env.sample`; `ApplicationItem.tsx`, `ProjectCard.tsx`, … |
| `REACT_APP_V5_PLATFORM_PREVIEW_URL` | Legacy v5 preview iframe base | `.env.sample`; `PreviewDevice.tsx` |
| `REACT_APP_DOCUMENTATION_URL` | Documentation link | `.env.sample`; `NavbarMain.tsx` |
| `REACT_APP_DEBUG_AGENT_URL` | Debug agent iframe/API origin | `.env.sample`; `Debug.tsx`, `ai-project/index.tsx`, … |
| `REACT_APP_NEW_CONNECTOR_URL` | OAuth connector callback base | `.env.sample`; `AIExternalApiDomainAuth.tsx` |
| `REACT_APP_SUPABASE_CLIENT_ID` | Supabase OAuth app client id | `.env.sample`; `SupabaseModal.tsx` |
| `REACT_APP_ORANGEKLOUD_COMPANY_ID` | Feature toggles / Orangekloud-only UI | `.env.sample`; `PluginInterface.tsx`, `PluginMarketplace.tsx` |
| `REACT_APP_BLOCK_AI_FEATURE` | Restrict AI to Orangekloud users when `true` | `.env.sample`; `Helper.ts` |
| `REACT_APP_DISABLE_STRICT_MODE` | React StrictMode off in dev/testing | `.env.sample`; `index.tsx` |

### Other (secondary)

| Variable / key | Purpose | Used in |
|---|---|---|
| `PORT`, `GENERATE_SOURCEMAP` | CRA build/dev (see CRA docs) | `frontend/.env.sample` |

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| Main backend | HTTP + OAuth redirects | Authentication, CRUD, proxy, notifications | `HttpRequest.ts`, `Redirect.ts`, zustand slices |
| Client backend | HTTP | AI build-related APIs | `clientBackendHttpRequest` in `HttpRequest.ts` |
| AI backend | HTTP (+ resolved VM host) | AI projects, documents, LB URL | `HttpRequest.ts`, `AiSlice.ts`, `Socket.ts` helpers |
| Service manager | HTTP | Plugins/packages | `serviceManagerHttpRequest` |
| Subscription service | HTTP + deep links | Billing | `subscriptionHttpRequest`, nav links |
| Storage / CDN | HTTP (URLs) | Assets and build artifacts | `REACT_APP_STORAGE_URL` usages |
| WebSocket service | WebSocket | Real-time updates | `Socket.ts` |
| SSO service | Browser navigation | Cookie/session invalidation | `REACT_APP_SSO_URL`, `Logout.tsx` |
| Supabase | OAuth client id only in FE | Connected Supabase OAuth flow | `SupabaseModal.tsx`, `SupabaseRedirectPage.tsx` |
| Debug agent UI | iframe + postMessage | In-browser AI debugging | `Debug.tsx`, `ai-project/index.tsx` |
| New connector | OAuth callback URL | External API registrations | `AIExternalApiDomainAuth.tsx` |

## 10. Main Flows

### Flow: OAuth login and session

1. User visits protected route → `ProtectedRoute` enforces cookie/session.
2. Unauthenticated navigation uses `/login` → `RedirectToSsoLogin` builds URL via `getOAuthLoginUrl()` (`Redirect.ts`) pointing at **main-backend** `{version}/{login_path}` with optional `state` (return URL).
3. OAuth callback lands on `/login/authorize` (`login/authorize`).
4. 401 responses on `httpRequest` clear user state and send user back toward login (`HttpRequest.ts`).
5. Optional `redirectSuccessfulAuthentication` / `redirectFailedAuthentication` for token-based redirects (`Redirect.ts`).

### Flow: AI project workspace and debug

1. User enters `/ai/...` routes guarded by company + AI subscription guards (`Route.tsx`).
2. Project pages load AI state via **AI backend** and related APIs (`AiSlice.ts`, hooks).
3. Debug view embeds **debug agent** in an iframe using `REACT_APP_DEBUG_AGENT_URL`; parent/child coordinate via **`postMessage`** (`ai-project/index.tsx`, `Debug.tsx`).
4. Load balancer VM URL fetched from AI backend for per-project infra (`getLoadBalancerUrl` in `HttpRequest.ts`).

### Flow: Marketplace and applications

1. Public marketplace routes (`/marketplace/*`) vs authenticated dashboard/projects (`/` , `/project`, `/my-projects`).
2. Application lifecycle uses **main-backend** endpoints in `ApplicationSlice.ts` plus storage URLs from `Helper.ts` / env.
3. **Implementation control** posts validate limits (`ImplementControl.ts`).

### Flow: Subscription / billing UX

1. Links to `{REACT_APP_SUBSCRIPTION_URL}/dashboard` and credits URL from navbar / modals (`NavbarMain.tsx`, `AiNavbar.tsx`).
2. Some subscription API calls via `subscriptionHttpRequest` (`UserSlice.ts`, login authorize flow).

### Flow: Supabase OAuth (AI create project)

1. User completes flow in modal; redirects to **`/supabase/callback`** (`Route.tsx`).
2. Window messaging back to opener (`SupabaseRedirectPage.tsx`).

## 11. Things Other Repos May Depend On

- **Stable client routes** (deep links): e.g. `/login`, `/login/authorize`, `/supabase/callback`, `/marketplace/*`, `/ai/*`, `/admin/*`. Defined in `frontend/src/Route.tsx`.
- **OAuth redirect URI shape**: Consumers must align with **`getOAuthLoginUrl()`** and callback handling on **`/login/authorize`** (`Redirect.ts`, `Route.tsx`).
- **`postMessage` contracts** with embedded **debug agent** iframe (`Debug.tsx`): message types include e.g. `MAIN_FRONTEND_DEBUG_IS_READY`, `MAIN_FRONTEND_DEBUG_IS_LEAVING`, `DEBUG_AGENT_STOPPED`, and handshake with `event.origin === REACT_APP_DEBUG_AGENT_URL`.
- **`postMessage` from Supabase callback** popup to opener (`SupabaseRedirectPage.tsx`).
- **`window.postMessage`** in plugin/connect flows (`PluginInterface.tsx`, `PluginSettingsPanel.tsx`) — INFERRED iframe/plugin integration surfaces; payload contracts should be coordinated with sibling repos.
- **Token query redirect**: Optional `redirectUrl?token=` pattern in `redirectSuccessfulAuthentication` (`Redirect.ts`).
- **`Api-Key` header** alongside Bearer for portions of **main-backend** (`ApplicationSlice.ts`, `NotificationSlice.ts`).
- **`v5-proxy-request`** contract for external HTTP through **main-backend** cookies endpoint (`Request.ts`).