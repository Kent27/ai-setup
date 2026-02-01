You are generating **human-facing repository documentation** intended to help new and returning developers understand, run, and safely modify this codebase.

## Primary Goal
Create clear, practical documentation that explains **what this repository is**, **how it is structured**, and **how developers work with it**, **without duplicating or contradicting `AGENTS.md`**.

This documentation is **not** a replacement for `AGENTS.md`.  
`AGENTS.md` defines **AI behavior and contribution rules**.  
This documentation explains **the system itself**.

---

## Scope & Boundaries (Important)
- ❌ Do NOT restate coding rules, AI instructions, or contribution policies already defined in `AGENTS.md`.
- ❌ Do NOT give instructions like “the agent should…” or “always do X when editing”.
- ✅ You MAY reference `AGENTS.md` when behavior or rules are intentionally delegated there.
- ✅ Focus on **understanding**, not enforcement.

If potential overlap with `AGENTS.md` is unclear, insert a short inline note like:
> _Note: If this overlaps with AGENTS.md, defer to AGENTS.md as the source of truth._

---

## What This Documentation MUST Cover

### 1. Repository Overview
- What problem this repository solves
- Who it is for (e.g. backend service, web app, internal tool)
- High-level responsibilities and non-goals

### 2. Architecture & Mental Model
- High-level architecture (components, services, or layers)
- How data and control flow through the system
- External dependencies (e.g. databases, APIs, queues)
- Include a simple diagram (ASCII or Mermaid) if it improves clarity

### 3. Codebase Structure
Explain the **main directories and files**, including:
- Where core logic lives
- Where configuration lives
- Where integrations or adapters live
- Which areas are most commonly modified

For each major folder:
- What it contains
- Why it exists
- When a developer would touch it

### 4. Key Design Decisions
Document important **design choices and trade-offs**, such as:
- Why a certain framework, pattern, or service was chosen
- Constraints that shaped the design
- Assumptions that future developers should know

Keep this concise and factual.  
If a full rationale exists elsewhere, link to it.

### 5. Common Developer Workflows
Provide practical, step-by-step guidance for common tasks, such as:
- Running the project locally
- Adding a new feature or module
- Modifying an existing flow
- Debugging or tracing behavior

Prefer **checklists or numbered steps** over long prose.

### 6. Onboarding Notes
Include anything a new developer would struggle to infer:
- Required environment variables or setup assumptions
- External systems they must have access to
- “Gotchas” or sharp edges worth knowing early

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

---

## Output Expectations
Produce clean Markdown documentation suitable for files such as:
- `README.md`
- `docs/ARCHITECTURE.md`
- `docs/WORKFLOWS.md`

Ensure:
- No duplication or contradiction with `AGENTS.md`
- Clear separation between **system understanding** (this doc) and **agent behavior** (`AGENTS.md`)
- Content is immediately useful to future developers

Confirm that this documentation:
- Explains the repository clearly
- Avoids policy or AI-instruction overlap
- Complements `AGENTS.md` instead of competing with it
