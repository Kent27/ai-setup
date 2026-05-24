# Create or Update Service Map Prompt

You are updating the repository documentation hub named `repo-hub`.

Target file:

`repo-hub/maps/service-map.md`

If the `repo-hub/maps/` folder does not exist, create it.

If the target file does not exist, create it.

If the target file already exists, update it carefully.

## Source of Truth

Use only information already documented in:

- `repo-hub/repo-profiles/*.md`
- `repo-hub/flows/*.md` if relevant

## Task

Create or update a high-level service map that explains how repositories/services connect.

The service map should help developers and AI agents quickly understand:

1. What each repo/service does
2. What each repo/service owns
3. What each repo/service exposes
4. What each repo/service calls
5. Which important flows each repo/service participates in
6. Which relationships are confirmed, inferred, or unclear

## Update Rules

If `repo-hub/maps/service-map.md` already exists:

- Preserve useful existing structure and manually written notes.
- Add missing information from repo profiles.
- Correct outdated information when repo profiles clearly contradict it.
- Do not remove useful information unless it is clearly obsolete or contradicted.
- Keep the document concise and high-level.
- Keep the **Exposes** column to short summaries; do not paste full endpoint catalogs (those live in each `repo-hub/repo-profiles/*.md`).
- Do not duplicate detailed table/model information from `database-map.md`.
- Mark uncertainty clearly.

## Required Output Format

Use this exact structure:

```md
# Service Map

## Overview

Briefly explain what this service map covers.

## Services

| Repo / Service | Purpose | Owns | Exposes | Calls / Depends On | Main Flows |
|---|---|---|---|---|---|
| `<repo-name>` | ... | ... | ... | ... | ... |

## Service Relationships

| From | To | Relationship | Evidence / Source | Confidence |
|---|---|---|---|---|
| `<source-repo>` | `<target-repo>` | ... | ... | High / Medium / Low |

## Important Cross-Repo Flows

| Flow | Repos Involved | Summary | Detailed Doc |
|---|---|---|---|
| ... | ... | ... | `repo-hub/flows/<flow-file>.md` |

## Known Gaps / Needs Confirmation

- ...
```

## Field Guidance

### `Services` table

Use one row per repo/service.

- `Repo / Service`: repo or service name
- `Purpose`: short description of what it does
- `Owns`: main domain/data/business responsibility
- `Exposes`: summarized endpoints/interfaces only
- `Calls / Depends On`: other repos/services it calls or relies on
- `Main Flows`: important flows this repo participates in

### `Service Relationships` table

Use this for confirmed or likely service-to-service connections.

Examples:

- frontend calls backend API
- subscription service calls main-backend
- gateway calls rate-limit endpoint
- backend uses Redis
- backend uses database

### `Important Cross-Repo Flows` table

Only include flows documented or clearly referenced in repo profiles or `repo-hub/flows/*.md`.

## Rules

- Treat `repo-hub/repo-profiles/*.md` as the main source of truth.
- Do not invent relationships.
- If a relationship is inferred, write `INFERRED` in the `Evidence / Source` column.
- If information is missing, write `NEEDS CONFIRMATION`.
- If confidence is unclear, use `Low`.
- Keep this map high-level.
- Detailed endpoint paths remain in each repo’s profile (`repo-hub/repo-profiles/*.md`).
- Detailed database/table/model information belongs in `repo-hub/maps/database-map.md`.
- Keep wording practical for future developers and AI coding agents.