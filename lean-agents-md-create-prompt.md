# Task: Generate Hierarchical AGENTS.md System

## What AGENTS.md Is For

AGENTS.md files are read by AI agents, not humans. Their only job is to prevent wasted turns by surfacing information the agent cannot infer from reading the code. Every line that fails this test consumes context budget without improving outcomes.

**The primary filter** — before including any information, ask: *"Would an agent get this wrong within 1-2 tool calls?"* If no, cut it.

---

## What Belongs vs. What Doesn't

| INCLUDE                                                        | EXCLUDE                                          |
|----------------------------------------------------------------|--------------------------------------------------|
| Non-obvious module boundaries and import rules                 | File trees (agents use `ls`)                     |
| Security guardrails and auth flow specifics                    | Framework conventions agents already know        |
| Dev commands only if non-standard (e.g. one command runs two servers) | Standard `npm test`, `pnpm build`          |
| Custom protocols or tags agents would misread as plain text    | Code style rules enforced by linters             |
| Gotchas that have caused real agent or developer mistakes      | "Definition of Done" checklists                  |
| JIT pointers to sub-files with a description of what each covers | grep/find commands                             |

---

## Phase 1: Repository Analysis

Analyze the codebase and produce a short structured summary covering:

1. Repository type (monorepo, multi-package, single project)
2. Tech stack (languages, frameworks, package manager)
3. Directories that have non-obvious conventions (candidates for sub-AGENTS.md)
4. Rules an agent would violate without being told
5. Gotchas — patterns that have caused real mistakes

Apply the primary filter to every candidate item before proceeding.

---

## Phase 2: Root AGENTS.md

**Ceiling: under 120 lines.** If you exceed this, you are documenting instead of routing. Re-apply the primary filter and cut everything that fails.

### Sections

**Project Snapshot** — required
One line per item. Only include fields that apply to this repo. Omit fields like "Styling" or "State" for backend-only projects.

```markdown
## Project Snapshot

- **Type**: [Monorepo / Single app / etc.]
- **Stack**: [Framework + language + key tools]
- **Package Manager**: [Tool + version]
```

**Shared Rules** — include if there are architectural rules an agent would violate
Focus on module boundaries, import conventions, and non-obvious runtime behavior. Omit anything a linter or type-checker enforces automatically.

```markdown
## Shared Rules

- Use `~/` imports inside `app/`; relative imports in app code fail lint.
- Treat `.server/` modules as server-only. Do not import them from client components.
- `pnpm dev` runs both Remix and the Express sidecar concurrently — not just the frontend.
```

**Security & Auth** — include if there are secrets or a non-standard auth flow
Be specific. Generic "don't commit secrets" adds no value. Include the auth mechanism if it is non-obvious.

```markdown
## Security & Auth

- `app/config.js` and `express/.env` are gitignored; start from the `.sample` files.
- Auth is not cookie-only: a session token also arrives via parent iframe `postMessage`.
- Read `app/root.tsx` and `app/lib/external/auth.ts` before touching any auth flow.
```

**Project Gotchas** — highest priority, include if any exist
Each bullet must describe a failure mode, not just a rule. If a gotcha is obvious from reading the code, cut it.

```markdown
## Project Gotchas

- The chat pipeline uses `<customArtifact>` / `<customAction>` tags — application protocol, not HTML. Do not escape or strip them.
- `WORK_DIR` is intentionally an empty string; paths are project-root-relative by design.
- In server modules and route handlers, avoid `node:path`; Vite HMR resolves it to a browser polyfill and breaks dev flows.
```

**JIT Index** — required if sub-AGENTS.md files exist
Each entry must describe what the sub-file covers, not just link to it. Agents use this to decide whether to read a sub-file before acting.

```markdown
## JIT Index

- `app/components/` → [AGENTS.md](app/components/AGENTS.md) — chat/workbench UI, client-only boundaries, attachment gotchas
- `app/lib/` → [AGENTS.md](app/lib/AGENTS.md) — stores, LLM orchestration, runtime protocol, custom AI tools
- `express/` → [AGENTS.md](express/AGENTS.md) — scheduler sidecar, env-driven log cleanup
```

---

## Phase 3: Sub-Folder AGENTS.md Files

**Ceiling: under 90 lines.** If you exceed this, the directory likely needs further decomposition into nested sub-files rather than more content in one file.

**Do not generate a sub-AGENTS.md if the directory follows standard framework conventions with no surprises.**

### Sections

**Identity** — one sentence describing what the directory contains and its primary tool or pattern.

**Conventions** — rules specific to this directory that an agent would get wrong. Omit anything already in root AGENTS.md or inferrable from the code.

**Gotchas** — failure modes specific to this directory. Each bullet describes what goes wrong, not just the rule.

**Key Files** — optional. Only if there is a non-obvious pattern an agent must copy rather than invent.

```markdown
## Key Files

| File | Why it matters |
|------|----------------|
| `chat/BaseChat.tsx` | Reference for streaming chat UI pattern |
```

---

## Output Format

```
---
File: AGENTS.md (root)
---
[content]

---
File: [dir]/AGENTS.md
---
[content]
```

---

## Final Checklist

- [ ] Every line survives the primary filter: an agent would get this wrong without being told
- [ ] Root is under 120 lines; sub-files are under 90 lines
- [ ] No file trees, grep commands, framework explanations, or linter-enforced style rules
- [ ] No "Definition of Done" checklists
- [ ] Project Gotchas section exists at the root level if any real gotchas were found
- [ ] JIT Index entries describe what each sub-file covers, not just link to it
- [ ] No information is duplicated across files
- [ ] All referenced files actually exist in the codebase
- [ ] No aspirational rules — document what IS, not what SHOULD BE
