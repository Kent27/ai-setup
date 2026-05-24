# Repo Profile: autopilot-debug-agent

Last updated: 2026-05-06  
Confidence: High  

To regenerate or align structure with the shared template, see **`prompts/repo-profile-generate-prompt.md`** (output file in each repo: `docs/repo-profile.md`).

## 1. Purpose

AI-powered debugger and IDE-style assistant (**Emobiq Debug Agent**) embedded via iframe in the **`emobiq-main-frontend`** parent application (the shell that owns the debugger iframe). Users chat with models to inspect and edit project files, run previews, manage checkpoints/commits (via backend), attach context (files, endpoints, plugins, browser storage when enabled), and stream LLM-assisted actions. Monetization hooks call the separate **Autopilot** backend for budgets, charges, runs, checkpoints, documents, and integrations.

## 2. Tech Stack

| | |
|---|---|
| Language | TypeScript (strict, main app); JavaScript (Express sub-package) |
| Framework | Remix 2 + React 18 |
| Runtime | Primary target: Cloudflare Pages / Workers compatibility (`nodejs_compat` in `wrangler.toml`); local dev via Vite + Remix Cloudflare dev proxy |
| Package manager | pnpm (root); npm inside `express/` |
| Database | No first-party relational DB ORM/migrations found in-repo; persistence is backend APIs, in-browser stores, filesystem logs, optional MCP (e.g. Supabase) tooling |
| Other important dependencies | Vercel AI SDK (`ai`, `@ai-sdk/*`), Remix UnoCSS, Nanostores, isomorphic-git, Codemirror/xterm UI, Wrangler |

## 3. Important Folders / Files

| Path | Purpose |
|---|---|
| `app/routes/` | Remix routes and resource handlers (`api.*.ts` → `/api/...`; `preview.$id.$.ts` → `/preview/:id/*`) |
| `app/lib/.server/` | Server-only chat/LLM pipeline (context selection, streaming, tokens, transforms) |
| `app/lib/modules/llm/` | LLM provider registry and integrations (Anthropic, OpenAI-compat, Bedrock, Ollama, etc.) |
| `app/lib/external/` | HTTP clients: main backend auth, Autopilot backend, Service Manager plugins |
| `app/config.js` (+ `config.js.sample`) | Runtime URLs, API keys, feature flags (gitignored prod file) |
| `express/` | Sidecar Node server: cron-based log retention; historically used for heavier APIs (see codebase comments) |
| `wrangler.toml` | Cloudflare Pages build output dir and compatibility flags |
| `vite.config.ts` | Vite + Remix + UnoCSS; loads dotenv |

## 4. Exposed Endpoints

Remix conventional paths (file segments with `.` → URL `/`).

### UI / document routes

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/` | Debugger chat shell | `app/routes/_index.tsx` | Root loader tries `getAuthUser` (cookie → main backend); failure is non-fatal, client iframe auth may fill cookie |
| `GET` | `/chat/:id` | Same UI with chat id param | `app/routes/chat.$id.tsx` | Same as root (`getAuthUser` via root/layout behavior) |
| `GET` | `/session/timeout` | Session ended message | `app/routes/session.timeout.tsx` | None |
| `GET` | `/preview/:projectHex/*` | Served preview/asset content for iframe | `app/routes/preview.$id.$.ts` | Bearer token from request cookies (`Autopilot.fromRequest`) |

### API routes (`/api/*`)

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| `GET` | `/api/chat` | Preflight/auth for chat route | `app/routes/api.chat.ts` (loader `requireAuth`) | Yes (`requireAuth`) |
| `POST` | `/api/chat` | Main chat/action stream | `app/routes/api.chat.ts` (action) | Yes (`requireAuth` in loader; action invoked after auth flows) |
| `GET` / `POST` | `/api/models` | Model listing / operations | `app/routes/api.models.ts` | Yes (loader `requireAuth`) |
| `GET` / `POST` | `/api/llmcall` | Direct LLM call wrapper | `app/routes/api.llmcall.ts` | Yes |
| `GET` / `POST` | `/api/enhancer` | Prompt enhancement | `app/routes/api.enhancer.ts` | Yes |
| `GET` / `POST` | `/api/log` | Client/server logging plumbing | `app/routes/api.log.ts` | Yes |
| `GET` / `POST` | `/api/autopilot/import` | Project import via Autopilot | `app/routes/api.autopilot.import.ts` | Yes |
| `GET` | `/api/checkpoints/diff` | Diff for checkpoint | `app/routes/api.checkpoints.diff.ts` | Yes |
| `GET` | `/api/checkpoints/diff-files` | Diff file list | `app/routes/api.checkpoints.diff-files.ts` | Yes |
| `GET` / `POST` | `/api/checkpoints/commit` | Commit checkpoint | `app/routes/api.checkpoints.commit.ts` | Yes |
| `GET` / `POST` | `/api/checkpoints/revert` | Revert checkpoint | `app/routes/api.checkpoints.revert.ts` | Yes |
| `POST` | `/api/codes/transform-to-patch` | Transform LLM patch output for autopilot `/codes/patch` | `app/routes/api.codes.transform-to-patch.ts` | Conditional: optional `DEBUG_API_KEY` in `~/config`; if set, Bearer or `X-API-Key` |

### Express sidecar (`express/index.js`)

| Method | Path | Purpose | Main handler/file | Auth |
|---|---|---|---|---|
| — | — | **Current code defines no routers/routes.** Server boots CORS middleware, JSON body parser, listens on `PORT`, runs scheduler. | `express/index.js` | N/A |

`express/` formerly documented `/api/v1/tools/validate-html` with `html-validate`; that routing is **not present** in the current `express/index.js`, and **`html-validate` is no longer an Express dependency**.

## 5. Outbound API Calls

### Emobiq / internal backends (from `app/lib/external/` + `~/config`)

| Target Service / Host | Method | Endpoint / URL pattern | Purpose | Source file |
|---|---|---|---|---|
| Main backend | `POST` | `{MAIN_BACKEND_URL}/oauth/tokens/introspect` | Validate Bearer cookie, load user | `app/lib/external/auth.ts` |
| Service Manager API | `GET` | `{SERVICE_MANAGER_URL}/editor-data?...` | Plugin editor/function definitions | `app/lib/external/service-manager.ts` |
| Autopilot API | Mixed | `{AUTOPILOT_AI_URL}/projects/`, `{AUTOPILOT_AI_URL}/projects/{hex}/mode` … | Projects + mode | `app/lib/external/autopilot.ts` |
| Autopilot | Mixed | `{AUTOPILOT_AI_URL}/documents/*` (`release`, `download/*`, `upload`, `delete`, `image/download/*`) | Artifact/document IO | `app/lib/external/autopilot.ts` |
| Autopilot | Mixed | `{AUTOPILOT_AI_URL}/chats/*` (`history`, `mark-edited`, `human_input/{hex}`, etc.) | Chat history persistence | `app/lib/external/autopilot.ts` |
| Autopilot | Mixed | `{AUTOPILOT_AI_URL}/pricing/*` (`balance/{user}`, `expected_cost`, `apply_charge`) | Budget and billing | `app/lib/external/autopilot.ts` |
| Autopilot | Mixed | `{AUTOPILOT_AI_URL}/runs/{hex}/debug`, `…/stop` | Run lifecycle | `app/lib/external/autopilot.ts` |
| Autopilot | Mixed | `{AUTOPILOT_AI_URL}/checkpoints/{hex}/commit`, `rollback`, `diff-files`, `diff` | Checkpoint operations | `app/lib/external/autopilot.ts` |
| Autopilot | Mixed | `{AUTOPILOT_AI_URL}/integration/*`, `{AUTOPILOT_AI_URL}/prompts/git-diff`, `{AUTOPILOT_AI_URL}/codes/patch`, `{AUTOPILOT_AI_URL}/codes/lint`, `{AUTOPILOT_AI_URL}/progress/` | Plugins, codegen, lint | `app/lib/external/autopilot.ts` |

### LLM and other providers

| Target | Purpose | Evidence |
|---|---|---|
| Model vendor APIs | Chat/completions via Vercel AI SDK providers | Dependencies `@ai-sdk/*`, `ollama-ai-provider`, `@openrouter/ai-sdk-provider`, `amazon-bedrock`, etc.; `app/lib/modules/llm/` |
| Google Fonts | Font CSS | `app/root.tsx` link tags |

### Helpers

| `app/lib/fetch.ts` | Uses `fetch` or dev-time `node-fetch` with HTTPS agent override |
| `app/utils/network.ts` | `fetchWithRetry` used by Autopilot client |

**`AUTOPILOT_AI_URL` is configured including the `/api/v1` segment** (e.g. `{origin}/api/v1`); `app/lib/external/autopilot.ts` appends path segments directly after that base.

## 6. Database / Models / Tables

| Table / Model / Entity | Purpose | Read/Write | Source file |
|---|---|---|---|
| — | No in-repo SQL/ORM schema | — | — |

Conceptual **`AuthUser`** shape is typed in `app/lib/external/auth.ts` (from main backend introspection JSON). Persisted artifacts and chat rows live in **Autopilot** (server-side); this repo consumes them via HTTP only.

Optional **Supabase MCP** can be configured in `config.js.sample` (`MCP_SERVERS`) for agent tooling against a Supabase project — not a bundled app database.

## 7. Jobs / Queues / Cron / Workers

| Name | Type | Purpose | Source file |
|---|---|---|---|
| `cleanupOldLogs` | Cron (midnight daily, optional immediate run) | Delete rotated `YYYY-MM-DD.log` files older than `LOG_RETENTION_DAYS` under project `LOG_DIRECTORY` | `express/app/scheduler/scheduler.js` |

No separate queue worker process found besides this scheduler.

## 8. Configuration & environment

**Primary mechanism:** **`app/config.js`** (committed template: **`app/config.js.sample`**). Runtime integration URLs, secrets **names**, and most feature flags are read via `import config from '~/config'`, not `.env`. **`.env.example`** notes env is deprecated for main app integration settings (Vite may still load `.env` for `VITE_*` and tooling).

### `~/config` — integration-focused keys (non-exhaustive)

| Key | Purpose | Typical consumers |
|---|---|---|
| `AUTOPILOT_AI_URL` | Autopilot API base URL (includes `/api/v1` segment) | `app/lib/external/autopilot.ts` |
| `SERVICE_MANAGER_URL` | Plugin / editor-data API host | `app/lib/external/service-manager.ts` |
| `MAIN_BACKEND_URL` | Main backend OAuth / API host (e.g. `/oauth/tokens/introspect`) | `app/lib/external/auth.ts` |
| `MAIN_FRONTEND_URL` | Parent shell origin (`frame-ancestors`, `postMessage` allowlist references) | `app/root.tsx`, `app/entry.server.tsx`, UI `postMessage` targets |
| `DEBUG_API_KEY` | Optional Bearer / `X-API-Key` gate for `/api/codes/transform-to-patch` | `app/routes/api.codes.transform-to-patch.ts` |
| `IS_EXTENDED_THINKING`, `ANTHROPIC_EXTENDED_THINKING_*` | Extended-thinking / token budget for Claude | `app/lib/.server/llm/stream-text.ts` |
| `LLM_CONTEXT_OPTIMIZATION`, `MAX_FILES_IN_CONTEXT_BUFFER`, `ENABLE_PRIORITIZE_SELECT_CONTEXT_FILES` | Context selection and file buffer limits | `app/lib/.server/llm/select-context.ts`, prompts, chat route |
| `API_ENDPOINTS_CAP` | Cap on API endpoints injected into select-context | `app/lib/.server/llm/context-allocator.ts`, `select-context.ts`, `app/routes/api.chat.ts` |
| `MODEL_TOKEN_INFO` | Per-model context window hints for budgeting | `app/lib/.server/llm/token-counter.ts`; see also `docs/features/token-budget-guide.md` |
| `MCP_SERVERS` | Optional stdio MCP server definitions (e.g. Supabase) | MCP wiring; shape in sample |
| `ENABLE_BROWSER_STORAGE_CONTEXT`, `BROWSER_STORAGE_KEY_CAP`, `ENABLE_BROWSER_STORAGE_TOOL`, `ENABLE_BROWSER_STORAGE_TOOL_FOR_STANDARD`, `SHOW_BROWSER_STORAGE_TOOL_UI` | Browser storage in context / tool / UI | `app/lib/.server/llm/*`, chat route |
| `LOG_DIRECTORY`, `LOG_MIN_LEVEL`, `LOG_WRITE_*`, `LOG_RETENTION_DAYS` | Logging and Express scheduler retention | `app/utils/logger.ts`, `express/app/scheduler/scheduler.js` |

Full list and comments live in **`app/config.js.sample`**.

### Process / Vite / Express (outside `~/config`)

| Variable | Purpose | Used in |
|---|---|---|
| (optional `.env` via Vite `dotenv`) | Dev/build-time injection (`VITE_*`, etc.) | `vite.config.ts` |
| `VITE_GITHUB_ACCESS_TOKEN` | Fallback GitHub token for starter flows | `app/utils/selectStarterTemplate.ts` |
| `VITE_DISABLE_PERSISTENCE` | Disable IndexedDB chat persistence flag | `app/lib/persistence/useChatHistory.ts` |
| `DEFAULT_NUM_CTX` | Ollama `num_ctx` (default 32768 if unset) | `app/lib/modules/llm/providers/ollama.ts` |
| `RUNNING_IN_DOCKER` | Rewrites localhost Ollama / LMStudio base URLs | `app/lib/modules/llm/providers/ollama.ts`, `lmstudio.ts` |
| `PORT` | Express listen port | `express/index.js` |
| `CORS_WHITELIST` | Comma-separated origins for Express CORS | `express/index.js` |

## 9. Service Dependencies

| Dependency | Type | Why it is needed | Evidence |
|---|---|---|---|
| Main Emobiq backend | OAuth / API | Token introspection, user profile | `app/lib/external/auth.ts` |
| Autopilot backend | REST API | Projects, documents, chat history, billing, checkpoints, runs, codegen | `app/lib/external/autopilot.ts` |
| Service Manager API | REST | Cordova/editor plugin definitions | `app/lib/external/service-manager.ts` |
| LLM vendors / gateways | SaaS APIs | Inference | `@ai-sdk/*`, providers under `app/lib/modules/llm/providers/` |
| Cloudflare Pages | Hosting | Deploy target | `wrangler.toml`, `package.json` scripts |
| **`emobiq-main-frontend`** (parent SPA) | UI integration | Embeds this app; iframe + `postMessage` auth handshake (`MAIN_FRONTEND_URL` must match shell origin) | `app/root.tsx`, workbench/chat `postMessage` usage |
| External npm MCP (optional) | stdio MCP | Supabase docs/DB tooling | `config.js.sample` `MCP_SERVERS` |

## 10. Main Flows

### Flow: Iframe bootstrap and SSO

1. User opens debugger inside the **`emobiq-main-frontend`** shell; CSP `frame-ancestors` permits `MAIN_FRONTEND_URL`.
2. `root` loader validates cookie via main backend introspect (`getAuthUser`).
3. If unauthenticated, inline script listens for parent `AUTH_TOKEN` `postMessage` (origin allowlist includes `MAIN_FRONTEND_URL` and dev host), sets cookie, reloads.

### Flow: Chat turn with checkpoints and billing

1. Client posts to `/api/chat` (`requireAuth`).
2. Server builds context (`select-context`, allocations, prompts in `app/lib/.server/llm/`).
3. Streams model output; optionally calls Autopilot for runs, checkpoints, documents, billing (`Autopilot` in `autopilot.ts`).

### Flow: Embedded preview assets

1. Browser requests `/preview/:projectHex/...`.
2. Loader builds `Autopilot` from Bearer cookie and fetches document map / content (`getDisplayDocuments` and related helpers in `preview.$id.$.ts` chain).

### Flow: Scheduled log housekeeping

1. Starting Express imports `runScheduler()` after listen.
2. `cleanupOldLogs` runs on schedule/configured retention (`config.LOG_RETENTION_DAYS`, `LOG_DIRECTORY`).

## 11. Things Other Repos Depend On

- **`postMessage` contract** between **`emobiq-main-frontend`** (parent) and this iframe: types `AUTH_TOKEN`, `AUTH_REQUIRED`, `MAIN_FRONTEND_DEBUG_IS_READY`, `REFRESH_PAGE` (`app/root.tsx`, `session.timeout.tsx`, workbench/chat components referencing parent messages).
- **Cookie name** used for API token (`API_TOKEN` in `app/lib/external/auth.ts` — the parent and this origin must agree so loaders and `/api/*` auth work).

**Note:** Remix routes such as **`/api/chat`**, **`/api/checkpoints/*`**, **`/preview/...`** are **used first-party** by scripts inside this iframe (same origin), e.g. `app/components/chat/Chat.client.tsx`, `app/components/workbench/Preview.tsx`, `app/lib/stores/workbench.ts` — not called directly cross-origin by **`emobiq-main-frontend`**. The loader **`app/routes/api.models.ts`** exists; **no `fetch('/api/models')` (or equivalent) usage was found under `app/`** at documentation time (**NEEDS CONFIRMATION**: dead route vs external/tooling callers).

Autopilot REST usage is **outbound from this repo** (see §5 / §9), not something other repos integrate with *via* this codebase.
