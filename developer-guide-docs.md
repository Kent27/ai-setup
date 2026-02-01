# Documentation Generation Prompt

You are generating **human-facing repository documentation** intended to help new and returning developers understand, run, and safely modify this codebase.

## Primary Goal

Create clear, practical documentation that explains **what this repository is**, **how it is structured**, and **how developers work with it**, **without duplicating or contradicting `AGENTS.md`**.

This documentation is **not** a replacement for `AGENTS.md`.  
`AGENTS.md` defines **AI behavior and contribution rules**.  
This documentation explains **the system itself**.

---

## Scope & Boundaries (Important)

- ❌ Do NOT restate coding rules, AI instructions, or contribution policies already defined in `AGENTS.md`.
- ❌ Do NOT give instructions like "the agent should…" or "always do X when editing".
- ✅ You MAY reference `AGENTS.md` when behavior or rules are intentionally delegated there.
- ✅ Focus on **understanding**, not enforcement.

If potential overlap with `AGENTS.md` is unclear, insert a short inline note: _If this overlaps with AGENTS.md, defer to AGENTS.md as the source of truth._

---

## Documentation File Separation (Critical)

When generating multiple documentation files, **each file must have a single, distinct purpose**. Do NOT duplicate content across files.

### File Purposes

| File | Purpose | Answers |
|------|---------|---------|
| `README.md` | Entry point | "What is this? How do I get started?" |
| `docs/ARCHITECTURE.md` | System design | "How is this designed? How does data flow?" |
| `docs/WORKFLOWS.md` | How-to recipes | "How do I do X?" |

### Separation Rules

**README.md** (entry point - keep lean):
- Project overview (1-2 paragraphs max)
- Quick start commands (install, configure, run)
- Code conventions
- Links to other docs
- ❌ NO architecture diagrams (link to ARCHITECTURE.md)
- ❌ NO workflow guides (link to WORKFLOWS.md)

**docs/ARCHITECTURE.md** (system design):
- Architecture diagram
- Layer/component descriptions
- External dependencies
- Key runtime concepts
- Key files reference
- ❌ NO setup instructions (that's README)
- ❌ NO how-to guides (that's WORKFLOWS)

**docs/WORKFLOWS.md** (how-to recipes):
- Step-by-step guides for common tasks
- Debugging guides
- Common file locations for specific changes
- ❌ NO project overview (that's README)
- ❌ NO architecture explanation (that's ARCHITECTURE)

### Before Writing, Ask:

1. Does this content already exist in another file?
2. Which file's core purpose does this content serve?
3. Can I link instead of duplicating?

---

## What This Documentation MUST Cover

### 1. Repository Overview (README.md only)

- What problem this repository solves
- Who it is for (e.g. backend service, web app, internal tool)
- High-level responsibilities and non-goals

### 2. Architecture & Mental Model (ARCHITECTURE.md only)

- High-level architecture (components, services, or layers)
- How data and control flow through the system
- External dependencies (e.g. databases, APIs, queues)
- Include a simple diagram (ASCII or Mermaid) if it improves clarity

### 3. Codebase Structure (README.md - brief, ARCHITECTURE.md - detailed)

- README: Simple directory tree with one-line descriptions
- ARCHITECTURE: Detailed explanation of layers and their responsibilities

### 4. Key Design Decisions (ARCHITECTURE.md only)

Document important **design choices and trade-offs**, such as:
- Why a certain framework, pattern, or service was chosen
- Constraints that shaped the design
- Assumptions that future developers should know

### 5. Common Developer Workflows (WORKFLOWS.md only)

Provide practical, step-by-step guidance for common tasks, such as:
- Adding a new feature or module
- Modifying an existing flow
- Debugging or tracing behavior

Prefer **checklists or numbered steps** over long prose.

### 6. Setup Instructions (README.md only)

- Install commands
- Environment configuration
- Run commands
- Validation commands

### 7. Onboarding Notes / Gotchas (WORKFLOWS.md - debugging section)

Include anything a new developer would struggle to infer:
- Required environment variables or setup assumptions
- External systems they must have access to
- "Gotchas" or sharp edges worth knowing early

---

## Tone & Style Guidelines

- Be concise, explicit, and structured
- Prefer bullets and headings over paragraphs
- Explain **why things exist**, not just **what they do**
- Assume the reader is a competent developer but new to this repo

---

## Cross-References

- When rules or behavioral constraints apply, **reference `AGENTS.md` instead of repeating them**
- When deeper rationale exists, link to existing docs (e.g. ADRs, architecture notes)
- **Link between docs instead of duplicating** (e.g., README links to ARCHITECTURE for diagrams)

---

## Output Expectations

Produce clean Markdown documentation suitable for files such as:
- `README.md`
- `docs/ARCHITECTURE.md`
- `docs/WORKFLOWS.md`

Ensure:
- No duplication or contradiction with `AGENTS.md`
- **No duplication between README, ARCHITECTURE, and WORKFLOWS**
- Clear separation between **system understanding** (this doc) and **agent behavior** (`AGENTS.md`)
- Content is immediately useful to future developers

---

## Final Checklist

Before finalizing, verify:

- [ ] README.md has NO architecture diagrams (only links)
- [ ] README.md has NO workflow guides (only links)
- [ ] ARCHITECTURE.md has NO setup instructions
- [ ] WORKFLOWS.md has NO project overview
- [ ] Each piece of information appears in exactly ONE file
- [ ] Files link to each other where appropriate
