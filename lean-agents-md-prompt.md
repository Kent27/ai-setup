# Task: Generate Hierarchical AGENTS.md System

## Research-Backed Principles

Based on empirical research (Gloaguen et al. 2026, Li et al. 2026):

1. **Context files reduce success rates when they add unnecessary requirements** -- every instruction is a constraint the agent will try to honor, and some cost more than they save.
2. **Focused files with 2-3 sections outperform comprehensive documentation** -- token efficiency matters more than thoroughness.
3. **Only document what agents cannot infer from code** -- agents can read file trees, understand TypeScript strict mode, and discover patterns via grep. Don't restate the obvious.
4. **Compliance checklists (lint, test, build) add failure points** -- agents already know to validate; prescriptive "Definition of Done" lists cause cascading failures on simple tasks.

### What Belongs in AGENTS.md

| INCLUDE (high value)                        | EXCLUDE (agents infer these)              |
|---------------------------------------------|-------------------------------------------|
| Non-obvious conventions (import aliases, naming rules) | File tree / directory descriptions |
| Security guardrails (gitignored secrets, auth flow)    | What TypeScript strict mode means  |
| Build/test commands (only if non-standard)             | Standard framework conventions     |
| Gotchas that waste agent turns                         | Obvious patterns visible in code   |
| Pointer to example files for unusual patterns          | Exhaustive API/type documentation  |

---

## Phase 1: Repository Analysis

Before generating, analyze the codebase and identify:

1. **Repository type**: Monorepo, multi-package, or single project?
2. **Tech stack**: Languages, frameworks, package manager
3. **Directories needing their own AGENTS.md** (only those with non-obvious conventions)
4. **Non-inferrable conventions**: Things an agent would get wrong without being told
5. **Known gotchas**: Patterns that have caused agent (or developer) mistakes

Present this as a short structured summary before generating files.

**Critical filter**: For each piece of information you plan to include, ask: *"Would an agent figure this out within 1-2 tool calls by reading the code?"* If yes, omit it.

---

## Phase 2: Generate Root AGENTS.md

### Line Count Rules

- If an AGENTS.md file does NOT already exist:
  - Root AGENTS.md target: 50-80 lines max.
  - Sub-folder AGENTS.md target: 30-50 lines max each.

- If an AGENTS.md file ALREADY exists:
  - Ignore the previous maximum line count rules.
  - Existing AGENTS.md files may expand up to 150 lines when necessary.
  - Still prioritize brevity, token efficiency, and high-signal information only.
  - Do not add filler, exhaustive documentation, or obvious framework explanations.

This is a routing file, not documentation.

### Required Sections (3 only)

**1. Project Snapshot** (5-8 lines)
One-line answers only:
```markdown
## Project Snapshot

- **Type**: [Monorepo / Single app / etc.]
- **Stack**: [Framework + language + key tools]
- **Styling**: [Approach]
- **State**: [State management library]
- **Package Manager**: [Tool + version]
```

**2. Non-Obvious Conventions** (10-20 lines)
ONLY rules an agent would violate without being told:
```markdown
## Conventions

### Import Rules
- Use `~/` prefix for all imports (aliased to `./app/`). Relative imports (`../`) will fail lint.

### Security
- `app/config.js` contains secrets — gitignored. Use `app/config.js.sample` as template.
- Never log or commit API keys.
```

**3. JIT Index** (10-15 lines)
Pointers to sub-files, nothing else:
```markdown
## JIT Index

- `app/components/` → [see AGENTS.md](app/components/AGENTS.md)
- `app/lib/` → [see AGENTS.md](app/lib/AGENTS.md)
- `app/routes/` → [see AGENTS.md](app/routes/AGENTS.md)
```

### What NOT to put in root AGENTS.md
- File trees (agents use `ls`)
- Build/test commands if they're standard (`npm test`, `pnpm build`)
- Framework explanations
- "Definition of Done" checklists
- grep/find commands (agents know how to search)

---

## Phase 3: Generate Sub-Folder AGENTS.md Files

**Target: 30-50 lines max each for new files. Existing AGENTS.md files may expand up to 150 lines if needed.** Only for directories with non-obvious patterns.

**Do NOT generate a sub-AGENTS.md if the directory follows standard framework conventions with no surprises.**

### Required Sections (2-3 only)

**1. Identity** (2 lines)
```markdown
# [Directory] AGENTS.md

[One sentence: what this directory contains and its primary framework/tool]
```

**2. Conventions & Gotchas** (15-30 lines)
The core value section. Only include rules that:
- An agent would get wrong without being told
- Have caused real bugs or failed PRs
- Are specific to THIS directory (not universal)

```markdown
## Conventions

- Feature components: `[Feature]/[Feature].tsx` (not `[feature].tsx`)
- All UI primitives use CVA for variants — see `ui/button.tsx` for the pattern
- Client-only components MUST use `.client.tsx` suffix (Remix convention for no-SSR)

## Gotchas

- `useStore()` is from nanostores, not zustand — don't import from wrong package
- Preview routes proxy to iframe; never return HTML directly
```

**3. Key Files** (5-8 lines, optional)
Only for non-obvious patterns an agent should copy:
```markdown
## Key Files

| File | Why it matters |
|------|---------------|
| `chat/BaseChat.tsx` | Reference for chat UI pattern with streaming |
| `ui/IconButton.tsx` | CVA variant pattern to follow |
```

### What NOT to put in sub-AGENTS.md files
- Content already in root AGENTS.md
- Standard framework patterns the agent knows
- Exhaustive file listings
- grep commands
- Type definitions (agents read the actual types)

---

## Output Format

Generate files in this order:

```
---
File: `AGENTS.md` (root)
---
[content — 50-80 lines, routing + non-obvious conventions only]

---
File: `[dir]/AGENTS.md` (only for dirs with non-obvious patterns)
---
[content — 30-50 lines, conventions + gotchas only]

[...repeat for each qualifying directory...]
```

---

## Final Checklist

### Information Diet (most important)
- [ ] Every line answers: "Would an agent get this wrong without being told?" — if no, cut it
- [ ] New root AGENTS.md files are under 80 lines
- [ ] New sub-files are under 50 lines each
- [ ] Existing AGENTS.md files only expand beyond previous limits when the added information is genuinely high value
- [ ] No AGENTS.md exceeds 150 lines
- [ ] No file trees, directory descriptions, or framework explanations
- [ ] No "Definition of Done" or prescriptive validation checklists
- [ ] No duplicate information across files

### Structure
- [ ] Root links to all sub-AGENTS.md files
- [ ] Each sub-file covers only its directory's non-obvious patterns
- [ ] No orphaned sub-files (all linked from root)

### Accuracy
- [ ] All referenced files actually exist in the codebase
- [ ] Conventions reflect actual codebase patterns (verified by reading code)
- [ ] No aspirational rules — only document what IS, not what SHOULD BE
