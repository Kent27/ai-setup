# Task: Generate Repository Documentation (AGENTS.md + Human Docs)

## Overview

Generate a **single root AGENTS.md** and **human-facing documentation** that work together. The human docs serve as detailed references that AGENTS.md can point to, eliminating the need for multiple AGENTS.md files throughout the codebase.

### Documentation Structure

| File | Audience | Purpose |
|------|----------|---------|
| `AGENTS.md` | AI Agents | Lightweight rules, JIT commands, links to human docs |
| `README.md` | Humans + Agents | Entry point, setup, conventions |
| `docs/ARCHITECTURE.md` | Humans + Agents | System design, patterns, key files |
| `docs/WORKFLOWS.md` | Humans + Agents | How-to guides, debugging, common tasks |

### Key Principle

**Human docs replace sub-folder AGENTS.md files.** Instead of creating `apps/web/AGENTS.md`, `services/auth/AGENTS.md`, etc., document those patterns in `docs/ARCHITECTURE.md` and `docs/WORKFLOWS.md`. The root AGENTS.md links to these docs.

---

## Phase 1: Repository Analysis

Before generating any files, analyze the codebase and document:

1. **Repository type**: Monorepo, multi-package, or single project?
2. **Primary tech stack**: Languages, frameworks, key tools
3. **Directory structure**: Major directories and their purposes
4. **Build system**: Package manager, workspace tools (Turborepo, Lerna, etc.)
5. **Testing setup**: Test framework, test locations, coverage tools
6. **Key patterns**: Code organization, naming conventions, important examples

Present this as a **structured analysis** before generating files.

---

## Phase 2: Generate AGENTS.md (Root Only)

Create a **single lightweight AGENTS.md** (~100-150 lines) with these sections:

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
- Validation commands to run before completing
- Key behavioral rules (follow patterns, use codebase search)

**3. Quick Reference** (10-15 lines)
- Project type and tech stack (1-2 lines)
- Key paths summary (components, services, routes, etc.)

**4. JIT Index Commands** (10-20 lines)
Provide search commands, NOT content:
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

**5. Common Patterns** (5-10 lines)
Point to example files for key patterns:
```markdown
## Common Patterns

### Adding a new page
See: `app/dashboard/page.tsx`, `docs/WORKFLOWS.md#adding-a-page`

### Creating a service
See: `src/services/auth/auth.ts`, `docs/WORKFLOWS.md#creating-a-service`
```

### AGENTS.md Constraints

- [ ] Under 150 lines total
- [ ] No detailed explanations (link to human docs instead)
- [ ] Commands are copy-paste ready
- [ ] Every pattern reference points to a real file
- [ ] Links to human docs for detailed guidance

---

## Phase 3: Generate README.md

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
- pnpm 8+

### Setup
```bash
pnpm install
cp .env.example .env.local
pnpm dev
```
```

**3. Project Structure** (10-20 lines)
Simple directory tree with one-line descriptions:
```markdown
## Project Structure

```
├── app/              # Next.js app router pages
├── src/
│   ├── components/   # React components
│   ├── services/     # Business logic & API calls
│   ├── states/       # Zustand stores
│   └── types/        # TypeScript types
└── docs/             # Documentation
```
```

**4. Code Conventions** (10-15 lines)
- TypeScript configuration
- Styling approach
- Import conventions
- Commit message format

**5. Available Scripts** (5-10 lines)
```markdown
## Scripts

| Command | Description |
|---------|-------------|
| `pnpm dev` | Start development server |
| `pnpm build` | Production build |
| `pnpm lint` | Run ESLint |
| `pnpm test` | Run tests |
```

**6. Documentation Links** (3-5 lines)
```markdown
## Documentation

- [Architecture](docs/ARCHITECTURE.md) — System design and patterns
- [Workflows](docs/WORKFLOWS.md) — How-to guides
```

**7. PR Checklist** (5-10 lines)
```markdown
## PR Checklist

Before submitting:
- [ ] `pnpm lint` passes
- [ ] `pnpm build` succeeds
- [ ] Tests pass (if applicable)
- [ ] No console.log statements
```

### README.md Constraints

- [ ] No architecture diagrams (link to ARCHITECTURE.md)
- [ ] No workflow guides (link to WORKFLOWS.md)
- [ ] Keep setup instructions concise

---

## Phase 4: Generate docs/ARCHITECTURE.md

Create the system design documentation (~200-400 lines):

### Required Sections

**1. System Overview** (10-20 lines)
- Architecture diagram (ASCII or Mermaid)
- High-level component descriptions

**2. Layer Descriptions** (30-50 lines)
For each major layer/directory, document:
- Purpose and responsibilities
- Key files and their roles
- Patterns used

Example:
```markdown
## Layers

### Pages (`app/`)
Next.js App Router pages. Each route folder contains:
- `page.tsx` — Page component
- `layout.tsx` — Layout wrapper (optional)
- `loading.tsx` — Loading state (optional)

### Components (`src/components/`)
React components organized by feature:
- `ui/` — Reusable UI primitives (Button, Input, etc.)
- `auth/` — Authentication components
- `chat/` — Chat feature components
```

**3. Data Flow** (10-20 lines)
- How data moves through the system
- State management approach
- API communication patterns

**4. External Dependencies** (10-15 lines)
- Third-party services
- APIs consumed
- Infrastructure dependencies

**5. Key Files Reference** (20-40 lines)
Document critical files that agents/developers should understand:
```markdown
## Key Files

| File | Purpose |
|------|---------|
| `src/services/api/apiClient.ts` | Centralized API client with auth |
| `src/states/authState.ts` | Authentication state management |
| `src/middleware.ts` | Route protection middleware |
```

**6. Patterns & Conventions** (30-50 lines)
**This section replaces sub-folder AGENTS.md files.**

Document patterns by domain with file examples:
```markdown
## Patterns

### Components
- Use functional components with TypeScript
- Props interface above component: see `src/components/ui/button.tsx`
- Colocate styles when component-specific

### Services
- Export async functions, not classes
- Handle errors with try/catch, return typed results
- Example: `src/services/auth/auth.ts`

### State (Zustand)
- One store per domain
- Use selectors for derived state
- Example: `src/states/authState.ts`
```

**7. Design Decisions** (10-20 lines)
Document important choices and trade-offs:
```markdown
## Design Decisions

### Why Zustand over Redux
- Simpler API, less boilerplate
- Better TypeScript inference
- Sufficient for current scale

### Why App Router over Pages Router
- Server components reduce client bundle
- Better data fetching patterns
- Future-proof architecture
```

### ARCHITECTURE.md Constraints

- [ ] No setup instructions (that's README)
- [ ] No step-by-step workflows (that's WORKFLOWS)
- [ ] Every pattern has a real file example
- [ ] Includes diagram for system overview

---

## Phase 5: Generate docs/WORKFLOWS.md

Create the how-to guide documentation (~150-300 lines):

### Required Sections

**1. Common Tasks** (50-100 lines)
Step-by-step guides for frequent development tasks:

```markdown
## Adding a New Page

1. Create route folder: `app/app/[feature]/page.tsx`
2. Add page component (copy from `app/app/dashboard/page.tsx`)
3. If protected, ensure middleware covers the route
4. Add navigation link if needed

## Adding a New Service Function

1. Create/update file in `src/services/[domain]/`
2. Export async function with typed parameters and return
3. Use `apiFetch` for API calls (see `src/services/api/apiClient.ts`)
4. Add error handling with try/catch

## Adding a New Component

1. Create file in `src/components/[feature]/ComponentName.tsx`
2. Define Props interface
3. Export function component
4. Add to barrel export if exists (`index.ts`)
```

**2. Debugging Guide** (20-40 lines)
```markdown
## Debugging

### API Errors
1. Check network tab for response details
2. Verify auth token in request headers
3. Check `src/services/api/apiClient.ts` for error handling

### State Issues
1. Use React DevTools to inspect Zustand stores
2. Check state updates in `src/states/[domain]State.ts`
3. Verify selectors return expected values

### Build Failures
1. Run `pnpm lint` to check for syntax errors
2. Run `pnpm build` locally to reproduce
3. Check for missing dependencies or imports
```

**3. Common File Locations** (10-20 lines)
```markdown
## Where to Find Things

| I need to... | Look in... |
|--------------|------------|
| Add a new page | `app/app/[feature]/page.tsx` |
| Add a UI component | `src/components/ui/` |
| Add business logic | `src/services/[domain]/` |
| Add global state | `src/states/[domain]State.ts` |
| Add types | `src/types/[domain].ts` |
```

**4. Gotchas & Tips** (10-20 lines)
```markdown
## Gotchas

- **Environment variables**: Client-side vars need `NEXT_PUBLIC_` prefix
- **Imports**: Use `@/` for absolute imports from `src/`
- **API calls**: Always use `apiFetch`, never raw `fetch`
- **Auth state**: Access via `useAuthState()` hook, not direct import
```

### WORKFLOWS.md Constraints

- [ ] No project overview (that's README)
- [ ] No architecture explanation (that's ARCHITECTURE)
- [ ] Practical, actionable steps
- [ ] File paths are real and accurate

---

## Output Format

Generate files in this order:

```
---
File: `AGENTS.md`
---
[content]

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

### No Duplication
- [ ] Each piece of information appears in exactly ONE file
- [ ] Files link to each other instead of repeating content
- [ ] AGENTS.md links to human docs, not vice versa

### AGENTS.md Quality
- [ ] Under 150 lines
- [ ] JIT commands use real patterns from codebase
- [ ] All file references exist
- [ ] Commands are copy-paste ready

### Human Docs Quality
- [ ] README has no architecture diagrams or workflows
- [ ] ARCHITECTURE has no setup instructions
- [ ] WORKFLOWS has no project overview
- [ ] All file examples are real

### Agent Usability
- [ ] Human docs provide enough detail for agents to follow patterns
- [ ] ARCHITECTURE.md patterns section replaces need for sub-folder AGENTS.md
- [ ] WORKFLOWS.md provides step-by-step guidance agents can follow
