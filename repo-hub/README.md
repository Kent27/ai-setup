# repo-hub

Cross-repository documentation for selected Orangekloud / eMOBIQ services: a lightweight hub for humans and AI agents. It does **not** replace canonical docs (`README.md`, `AGENTS.md`, `docs/*`) inside each application repo.

## What lives here

| Path | Purpose |
|------|---------|
| [`repo-profiles/`](repo-profiles/) | One Markdown profile per in-scope repo (APIs, config, deps, flows). |
| [`flows/`](flows/) | End-to-end cross-repo narratives (sequences, endpoints, data, failures). |
| [`maps/`](maps/) | Aggregated views: [`service-map.md`](maps/service-map.md), [`database-map.md`](maps/database-map.md). |
| [`prompts/`](prompts/) | Reusable prompts to regenerate hub content safely and consistently. |

## How to use it (readers)

1. Open `repo-profiles/` for services touched by your change.
2. Use `flows/` when you care about ordering, endpoints, and config across repos.
3. Use `maps/` for a quick topology or datastore overview.

Before changing application code, read each repo’s own `README.md`, `AGENTS.md`, and `docs/` as appropriate.

## Developer SOP — keeping repo-hub up to date

Follow this **after completing a substantive ticket** or **at the close of an epic** when work touches an application repo that belongs in [`repo-profiles/`](repo-profiles/).

**Goals.** Onboarding stays fast for humans and agents; cross-repo flows, service wiring, and data stores stay in one predictable place.

### 1. Create or update the profile in the application repo

In the repository where you shipped the change:

1. Open or create **`docs/repo-profile.md`** (canonical location in that repo).
2. Use [`prompts/create-or-update-repo-profile.md`](prompts/create-or-update-repo-profile.md) with your AI tooling (or the checklist inside it) to refresh the profile from the codebase.
3. Commit **`docs/repo-profile.md`** in **that application repo** as usual. (**Exception:** `emobiq-migration` may also update **`docs/erd.md`** per that prompt.)

If the hub does not yet list that repo, add **`repo-profiles/<repo-name>.md`**.

### 2. Copy the profile into this repo (`repo-hub`)

1. Copy the finalized **`docs/repo-profile.md`** from the application repo once it is ready there.
2. Paste into **`repo-profiles/<repo-name>.md`** using the git **repository slug** (examples: `emobiq-main-frontend.md`, `autopilot-ai.md`, `orangekloud-sso.md`). Match an existing sibling or [`maps/service-map.md`](maps/service-map.md) if unsure.
3. Land the changes in **repo-hub**, keeping edits focused on profiles, maps, and flows where you can.

### 3. Cascading prompts (when more than one repo or contract moves)

Profiles drive maps and flows. After step 2, run **additional prompts** here when applicable.

**[`prompts/create-or-update-service-map.md`](prompts/create-or-update-service-map.md)** — update [`maps/service-map.md`](maps/service-map.md) when service boundaries or wiring change: new outbound deps, notable new consumers or route groups, joining or leaving a cross-repo flow. Source of truth: `repo-profiles/` and `flows/` (see prompt).

**[`prompts/create-or-update-database-map.md`](prompts/create-or-update-database-map.md)** — update [`maps/database-map.md`](maps/database-map.md) when durable state changes: new or renamed tables/stores/blobs/queues, or a new owning service. Source of truth: `repo-profiles/` and `flows/` (prompt: do not scan app code unless instructed).

**[`prompts/create-or-update-flow.md`](prompts/create-or-update-flow.md)** — add or update **`flows/<flow-slug>.md`** for important end-to-end paths (auth, payments/credits, build/compile, AI project generation, etc.). Source of truth: `repo-profiles/`, existing `flows/`, `maps/` as supporting detail (see prompt).

For small single-repo bugfixes with no contract change, **skip §3**; when in doubt, update the smallest map or flow that captures what you changed.

### Checklist (ticket / epic)

- [ ] Updated **`docs/repo-profile.md`** in the application repo.
- [ ] Copied/synced **`repo-profiles/<repo-name>.md`** in repo-hub.
- [ ] Integrations changed → ran **service-map** prompt → updated [`maps/service-map.md`](maps/service-map.md).
- [ ] Durable state changed → ran **database-map** prompt → updated [`maps/database-map.md`](maps/database-map.md).
- [ ] Cross-repo behaviour changed → ran **flow** prompt → updated **`flows/*.md`**.

### Prompt index

| Prompt | Typical use |
|--------|--------------|
| [`prompts/create-or-update-repo-profile.md`](prompts/create-or-update-repo-profile.md) | Build/update **`docs/repo-profile.md`** **inside each app repo**. |
| [`prompts/create-or-update-service-map.md`](prompts/create-or-update-service-map.md) | Refresh [`maps/service-map.md`](maps/service-map.md). |
| [`prompts/create-or-update-database-map.md`](prompts/create-or-update-database-map.md) | Refresh [`maps/database-map.md`](maps/database-map.md). |
| [`prompts/create-or-update-flow.md`](prompts/create-or-update-flow.md) | Create/update `flows/<flow-slug>.md`. |

## Source of truth

If repo-hub disagrees with a repo’s docs or running code, **trust that repository first**, then refresh this hub using the SOP above. Do not treat hub-only wording as overriding production behaviour.

## Out of scope

Repositories not listed under [`repo-profiles/`](repo-profiles/) are not covered by this hub (some product lines may be intentionally omitted).
