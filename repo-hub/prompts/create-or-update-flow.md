# Create or Update Flow Doc Prompt

You are documenting a cross-repo system flow in the repository documentation hub named `repo-hub`.

Target file:

`repo-hub/flows/<flow-slug>.md`

If the target file does not exist, create it.

If the target file already exists, update it carefully.

## User-Provided Flow

Flow name:

`<FLOW_NAME>`

Short description / known context:

`<SHORT_DESCRIPTION_OR_KNOWN_CONTEXT_FROM_USER>`

Relevant repos, if known:

`<RELEVANT_REPOS_OR_UNKNOWN>`

## Source of Truth

Use information from:

- `repo-hub/repo-profiles/*.md`
- existing `repo-hub/flows/*.md` if relevant
- `repo-hub/maps/*.md` if relevant

If instructed to inspect source code directly, use the repositories as supporting evidence.

## Task

Create or update a factual flow document that explains how this flow works end-to-end.

Focus on:

1. What triggers the flow
2. Which repos/services are involved
3. Each repo’s responsibility
4. End-to-end steps in order
5. Endpoints called
6. Database tables/models touched
7. Queues, workers, cron jobs, Redis keys, files, or storage involved
8. Config/env vars needed
9. Failure cases and expected behavior
10. Debugging notes
11. Unknowns / needs confirmation

## Update Rules

If the flow doc already exists:

- Preserve useful existing structure and manually written notes.
- Add missing information from repo profiles, maps, and relevant existing flow docs.
- Correct outdated information when the source docs clearly contradict it.
- Do not remove useful information unless it is clearly obsolete or contradicted.
- Preserve exact endpoint paths, method names, table names, config names, and file paths when available.
- Mark uncertainty clearly.
- Do not invent repos, endpoints, tables, configs, queues, or flow steps.

## Required Format

Use this exact structure:

```md
# <Flow Name>

## Purpose

Short explanation of what this flow does.

## Known Context

User-provided context and assumptions.

## Related Docs

- Service map: `repo-hub/maps/service-map.md`
- Database map: `repo-hub/maps/database-map.md`

## Repositories Involved

### `<repo-name>`

- Repo profile: `repo-hub/repo-profiles/<repo-name>.md`
- Responsibilities:
- Key files:

## End-to-End Flow

1. ...
2. ...
3. ...

## Endpoints

| From | To | Method | Endpoint | Purpose | Auth / Headers | Confidence |
|---|---|---|---|---|---|---|
| ... | ... | ... | ... | ... | ... | High / Medium / Low |

## Data / State

| Repo | Table / Model / Redis / Queue / Storage / File | Purpose | Read / Write | Confidence |
|---|---|---|---|---|
| ... | ... | ... | ... | High / Medium / Low |

## Config

| Repo | Config / Env Var | Purpose | Confidence |
|---|---|---|---|
| ... | ... | ... | High / Medium / Low |

## Failure Cases

| Case | Expected Behavior | Where Handled | Confidence |
|---|---|---|---|
| ... | ... | ... | High / Medium / Low |

## Debugging Notes

- ...

## Unknowns / Needs Confirmation

- ...
```

## Rules

- Prefer facts found in `repo-hub/repo-profiles/*.md`.
- Use `repo-hub/maps/*.md` only as supporting summaries, not as the main source of truth.
- If something is inferred, mark it as `INFERRED`.
- If something is uncertain, mark it as `NEEDS CONFIRMATION`.
- Do not invent endpoints, tables, config values, queues, storage paths, or repo relationships.
- Keep the document practical for future AI agents and developers.
- After creating or updating the flow, suggest updates needed for:
  - `repo-hub/maps/service-map.md`
  - `repo-hub/maps/database-map.md`