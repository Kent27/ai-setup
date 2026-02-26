# Task: Generate Human-Facing Documentation

## Overview

Generate developer documentation that complements (but is separate from) the AGENTS.md system. These files are for humans onboarding, debugging, and making architectural decisions.

**Do NOT duplicate content from AGENTS.md files.** AGENTS.md covers agent-specific conventions; these docs cover human understanding.

---

## Phase 1: Repository Analysis

Before generating, review:

1. **Tech stack, build system, deployment target**
2. **Setup steps** (prerequisites, env vars, install, run)
3. **Architecture layers** (how data flows, key abstractions)
4. **Common developer tasks** (add a feature, debug an issue, deploy)
5. **Known pain points** new developers hit

---

## Phase 2: Generate README.md

**Target: 100-150 lines.** The entry point -- setup and orientation only.

### Required Sections

**1. Project Overview** (5-10 lines)
What this project does, who it's for, high-level tech stack.

**2. Quick Start** (15-25 lines)
Prerequisites, install, env setup, run dev server. Copy-paste ready.

```markdown
## Quick Start

### Prerequisites
- Node.js 18+
- pnpm 9+

### Setup
```bash
pnpm install
cp app/config.js.sample app/config.js   # Edit with your keys
pnpm dev                                  # Runs on localhost:5173
```
```

**3. Project Structure** (10-15 lines)
Simple directory tree with one-line descriptions. Only top-level directories. Link to ARCHITECTURE.md for details.

**4. Available Scripts** (5-10 lines)
Table of build/dev/test/lint commands.

**5. Code Conventions** (5-10 lines)
Brief summary. Link to AGENTS.md for exhaustive rules (agents enforce them anyway).

**6. PR Checklist** (5-8 lines)
What must pass before merging.

**7. Documentation Links** (3 lines)
Links to ARCHITECTURE.md and WORKFLOWS.md.

### README Constraints
- [ ] No architecture diagrams (link to ARCHITECTURE.md)
- [ ] No step-by-step workflow guides (link to WORKFLOWS.md)
- [ ] Setup instructions are copy-paste ready and tested

---

## Phase 3: Generate docs/ARCHITECTURE.md

**Target: 200-350 lines.** System design and cross-cutting patterns.

### Required Sections

**1. System Overview** (15-25 lines)
Architecture diagram (ASCII or Mermaid) + high-level component descriptions.

**2. Layer Descriptions** (40-60 lines)
For each major layer: purpose, key files, how it connects to other layers.

**3. Data Flow** (15-25 lines)
How a request moves through the system. State management approach. API communication patterns.

**4. External Dependencies** (10-15 lines)
Third-party services, APIs, infrastructure.

**5. Cross-Cutting Patterns** (20-30 lines)
Authentication, error handling, logging -- patterns that span multiple directories.

**6. Design Decisions** (15-25 lines)
Important architectural choices and their trade-offs. Why X over Y.

### ARCHITECTURE Constraints
- [ ] No setup instructions (that's README)
- [ ] No step-by-step task guides (that's WORKFLOWS)
- [ ] Focuses on "why" and "how things connect", not "what files exist"

---

## Phase 4: Generate docs/WORKFLOWS.md

**Target: 150-250 lines.** Practical how-to guides for common tasks.

### Required Sections

**1. Common Tasks** (60-100 lines)
Step-by-step guides:
- Add a new page/route
- Add a new API endpoint
- Add a new component
- Add a new service/utility
- Modify the database schema (if applicable)

**2. Debugging Guide** (25-40 lines)
- How to debug common issues
- Where to find logs
- Common error messages and fixes

**3. "I need to..." Reference Table** (10-20 lines)
```markdown
| I need to...                | Look in...                        |
|-----------------------------|-----------------------------------|
| Add an API endpoint         | `app/routes/api.*.ts`             |
| Modify chat behavior        | `app/lib/.server/llm/`            |
| Add a UI component          | `app/components/ui/`              |
```

**4. Gotchas & Tips** (10-20 lines)
Things that trip up new developers.

### WORKFLOWS Constraints
- [ ] No project overview (that's README)
- [ ] No architecture explanation (that's ARCHITECTURE)
- [ ] Every guide is actionable with concrete file paths and commands

---

## Output Format

```
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

### No Duplication
- [ ] Each piece of information appears in exactly ONE file
- [ ] README doesn't contain architecture or workflows
- [ ] ARCHITECTURE doesn't contain setup or task guides
- [ ] WORKFLOWS doesn't contain project overview or design rationale
- [ ] None of these files duplicate AGENTS.md content

### Quality
- [ ] All file paths and commands reference real codebase artifacts
- [ ] Setup instructions are copy-paste ready
- [ ] Workflow guides have been verified against actual code structure
- [ ] No aspirational content -- only document what IS, not what SHOULD BE
