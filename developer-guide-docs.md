# Task: Generate Hierarchical AGENTS.md + Human Docs

## Overview

Generate a **hierarchical AGENTS.md system** and **human-facing documentation** that work together. This approach maximizes AI agent performance through token efficiency.

### Core Principles

1. **Root AGENTS.md is LIGHTWEIGHT** (~100-150 lines) - Universal guidance, links to sub-files
2. **Nearest-wins hierarchy** - Agents read the closest AGENTS.md to the file being edited
3. **JIT (Just-In-Time) indexing** - Provide paths/globs/commands, NOT full content
4. **Token efficiency** - Small, actionable guidance over encyclopedic documentation
5. **Sub-folder AGENTS.md files have MORE detail** - Specific patterns, examples, commands

### Documentation Structure

| File | Audience | Purpose |
|------|----------|---------|
| `AGENTS.md` (root) | AI Agents | Universal rules, links to sub-AGENTS.md files |
| `[directory]/AGENTS.md` | AI Agents | Directory-specific patterns, examples, commands |
| `README.md` | Humans + Agents | Entry point, setup, conventions |
| `docs/ARCHITECTURE.md` | Humans + Agents | System design, cross-cutting patterns |
| `docs/WORKFLOWS.md` | Humans + Agents | How-to guides, debugging, common tasks |

---

## Phase 1: Repository Analysis

Before generating any files, analyze the codebase and document:

1. **Repository type**: Monorepo, multi-package, or single project?
2. **Primary technology stack**: Languages, frameworks, key tools
3. **Major directories** that should have their own AGENTS.md:
   - Apps/pages (e.g., `app/`, `apps/web/`, `apps/api/`)
   - Components (e.g., `src/components/`)
   - Services/API (e.g., `src/services/`, `src/lib/`)
   - State management (e.g., `src/states/`, `src/store/`)
   - Workers/jobs (e.g., `workers/`, `jobs/`)
4. **Build system**: npm/pnpm/yarn? Workspaces? Turborepo?
5. **Testing setup**: Jest, Vitest, Playwright? Where are tests?
6. **Key patterns to document**:
   - Code organization patterns
   - Naming conventions
   - Critical files that serve as good examples
   - Anti-patterns to avoid

Present this as a **structured map** before generating any AGENTS.md files.

---

## Phase 2: Generate Root AGENTS.md

Create a **lightweight root AGENTS.md** (~100-150 lines max) with these sections:

### Required Sections

**1. Read First** (3-5 lines)
```markdown
## Read First

Before making changes, read these docs:
- `README.md` — setup, conventions, PR checklist
- `docs/ARCHITECTURE.md` — system design, layers, key files
- `docs/WORKFLOWS.md` — how to add features, debugging guides
```

**2. Agent Rules** (5-10 lines)
```markdown
## Agent Rules

1. **Always validate** before completing: `npm run lint && npm run build`
2. **Follow existing patterns** — copy from similar files, don't invent
3. **Use the codebase** — grep/find to discover patterns before writing
```

**3. Quick Reference** (10-15 lines)
```markdown
## Quick Reference

### Project Type
[Tech stack summary in 1-2 lines]

### Key Paths
- Routes: `app/**/page.tsx`
- Components: `src/components/`
- Services: `src/services/`
- State: `src/states/`
```

**4. JIT Index - Directory Map** (10-20 lines)
Link to sub-AGENTS.md files:
```markdown
## JIT Index

### Directory Structure
- App routes: `app/` → [see app/AGENTS.md](app/AGENTS.md)
- Components: `src/components/` → [see src/components/AGENTS.md](src/components/AGENTS.md)
- Services: `src/services/` → [see src/services/AGENTS.md](src/services/AGENTS.md)
- State: `src/states/` → [see src/states/AGENTS.md](src/states/AGENTS.md)
```

**5. JIT Commands** (5-10 lines)
```markdown
## JIT Index Commands

```bash
# Find a component
grep -rn "export.*ComponentName" src/components/

# Find a service function
grep -rn "export async function" src/services/

# Find page routes
find app -name "page.tsx"
```
```

**6. Common Patterns** (5-10 lines)
Point to example files:
```markdown
## Common Patterns

### Page with data fetching
See: `app/app/page.tsx`

### Service function
See: `src/services/organization/roles.ts`

### Zustand store
See: `src/states/authState.ts`
```

### Root AGENTS.md Constraints

- [ ] Under 150 lines total
- [ ] Links to all sub-AGENTS.md files
- [ ] No detailed explanations (delegate to sub-files or human docs)
- [ ] Commands are copy-paste ready

---

## Phase 3: Generate Sub-Folder AGENTS.md Files

For EACH major directory identified in Phase 1, create a **detailed AGENTS.md** (~50-100 lines):

### Required Sections

**1. Package Identity** (2-3 lines)
```markdown
# [Directory] AGENTS.md

What: [Brief description]
Tech: [Primary framework/tools for THIS directory]
```

**2. Patterns & Conventions** (20-40 lines)
**THIS IS THE MOST IMPORTANT SECTION**
- File organization rules
- Naming conventions
- Preferred patterns with **file examples**:

```markdown
## Patterns

### File Organization
- Feature components: `[Feature]/[Feature].tsx`
- UI primitives: `ui/[name].tsx`

### Do / Don't
- ✅ DO: Functional components with TypeScript like `ui/button.tsx`
- ❌ DON'T: Class components or inline styles
- ✅ Props: Define interface above component, see `auth/LoginContainer.tsx`
```

**3. Key Files** (5-10 lines)
```markdown
## Key Files

| File | Purpose |
|------|---------|
| `ui/button.tsx` | Button variants with CVA |
| `auth/LoginContainer.tsx` | Auth form pattern |
| `layout/OrgLayout.tsx` | Layout wrapper pattern |
```

**4. JIT Commands** (5-10 lines)
Directory-specific search commands:
```markdown
## Find Things

```bash
# Find component by name
grep -rn "export function" .

# Find component usage
grep -rn "import.*Button" ../..
```
```

**5. Gotchas** (3-5 lines, if applicable)
```markdown
## Gotchas

- Always use `@/` imports for absolute paths
- UI components use CVA for variants
- Wrap client components with `"use client"` directive
```

### Sub-Folder AGENTS.md Constraints

- [ ] Under 100 lines each
- [ ] No duplication with root AGENTS.md
- [ ] Every example references a real file in that directory
- [ ] Directory-specific patterns only

---

## Phase 4: Generate README.md

Create the entry point documentation (~100-200 lines):

### Required Sections

**1. Project Overview** (5-10 lines)
- What this project does
- Who it's for
- High-level tech stack

**2. Quick Start** (10-20 lines)
```markdown
## Quick Start

### Prerequisites
- Node.js 18+
- npm/pnpm

### Setup
```bash
npm install
cp .env.example .env.local
npm run dev
```
```

**3. Project Structure** (10-20 lines)
Simple directory tree with one-line descriptions.

**4. Code Conventions** (10-15 lines)
- TypeScript configuration
- Styling approach
- Import conventions
- Commit message format

**5. Available Scripts** (5-10 lines)
Table of npm scripts.

**6. Documentation Links** (3-5 lines)
Links to ARCHITECTURE.md and WORKFLOWS.md.

**7. PR Checklist** (5-10 lines)
What must pass before PR is ready.

### README.md Constraints

- [ ] No architecture diagrams (link to ARCHITECTURE.md)
- [ ] No workflow guides (link to WORKFLOWS.md)
- [ ] Keep setup instructions concise

---

## Phase 5: Generate docs/ARCHITECTURE.md

Create the system design documentation (~200-400 lines):

### Required Sections

**1. System Overview** (10-20 lines)
- Architecture diagram (ASCII or Mermaid)
- High-level component descriptions

**2. Layer Descriptions** (30-50 lines)
For each major layer, document purpose, key files, patterns.

**3. Data Flow** (10-20 lines)
- How data moves through the system
- State management approach
- API communication patterns

**4. External Dependencies** (10-15 lines)
Third-party services and APIs.

**5. Key Files Reference** (20-40 lines)
Critical files that developers should understand.

**6. Cross-Cutting Patterns** (20-30 lines)
Patterns that span multiple directories (authentication, error handling, etc.)

**7. Design Decisions** (10-20 lines)
Important choices and trade-offs.

### ARCHITECTURE.md Constraints

- [ ] No setup instructions (that's README)
- [ ] No step-by-step workflows (that's WORKFLOWS)
- [ ] Covers cross-cutting concerns; directory-specific patterns in sub-AGENTS.md

---

## Phase 6: Generate docs/WORKFLOWS.md

Create the how-to guide documentation (~150-300 lines):

### Required Sections

**1. Common Tasks** (50-100 lines)
Step-by-step guides for frequent development tasks.

**2. Debugging Guide** (20-40 lines)
How to debug common issues.

**3. Common File Locations** (10-20 lines)
"I need to... → Look in..." table.

**4. Gotchas & Tips** (10-20 lines)
Things new developers struggle with.

### WORKFLOWS.md Constraints

- [ ] No project overview (that's README)
- [ ] No architecture explanation (that's ARCHITECTURE)
- [ ] Practical, actionable steps

---

## Output Format

Generate files in this order:

```
---
File: `AGENTS.md` (root)
---
[content - lightweight, links to sub-files]

---
File: `app/AGENTS.md`
---
[content - routes/pages patterns]

---
File: `src/components/AGENTS.md`
---
[content - component patterns]

---
File: `src/services/AGENTS.md`
---
[content - service patterns]

---
File: `src/states/AGENTS.md`
---
[content - state management patterns]

[...additional sub-AGENTS.md as needed...]

---
File: `README.md`
---
[content]

---
File: `docs/ARCHITECTURE.md`
---
[content]

---
File: `docs/WORKFLOWS.md`
---
[content]
```

---

## Final Checklist

Before finalizing, verify:

### Hierarchy
- [ ] Root AGENTS.md links to all sub-AGENTS.md files
- [ ] Each major directory has its own AGENTS.md
- [ ] No orphaned sub-AGENTS.md (all linked from root)

### No Duplication
- [ ] Each piece of information appears in exactly ONE file
- [ ] Root doesn't repeat sub-file content
- [ ] Sub-files don't repeat each other
- [ ] AGENTS.md links to human docs, not vice versa

### AGENTS.md Quality
- [ ] Root under 150 lines
- [ ] Sub-files under 100 lines each
- [ ] JIT commands use real patterns from codebase
- [ ] All file references exist
- [ ] Commands are copy-paste ready
- [ ] Every "✅ DO" has a real file example

### Human Docs Quality
- [ ] README has no architecture diagrams or workflows
- [ ] ARCHITECTURE has no setup instructions
- [ ] WORKFLOWS has no project overview
- [ ] All file examples are real

### Agent Usability
- [ ] Agent editing `src/components/X.tsx` loads `src/components/AGENTS.md`
- [ ] Agent editing `src/services/Y.ts` loads `src/services/AGENTS.md`
- [ ] Nearest-wins provides relevant context with minimal tokens
