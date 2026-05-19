# Task: Update Existing AGENTS.md Files

## What AGENTS.md Is For

AGENTS.md files are read by AI agents. Their only job is to prevent wasted turns by surfacing information the agent cannot infer from reading the code. Every line that fails this test consumes context budget without improving outcomes.

**The primary filter** — before keeping or adding any information, ask: *"Would an agent get this wrong within 1-2 tool calls?"* If no, cut it.

---

## Rules

- Read each existing AGENTS.md before touching it.
- Make targeted edits only — do not rewrite or restructure unless the current structure actively misleads.
- Preserve all high-signal content. Remove only what is stale, duplicated, or fails the primary filter.
- Add only what an agent would get wrong without being told.

---

## What to Update

**Stale references** — file names, paths, or package versions that no longer exist in the codebase.

**Missing gotchas** — new patterns introduced since the file was last written that have caused or would cause agent mistakes. Each gotcha must describe the failure mode, not just the rule.

**Outdated conventions** — rules that no longer reflect how the codebase actually works. Remove them; do not replace with aspirational rules.

**JIT Index gaps** — new sub-directories with non-obvious patterns that need their own AGENTS.md and an entry in the root JIT Index. Each entry must describe what the sub-file covers, not just link to it.

**Bloat** — lines that were added over time that now fail the primary filter. Cut them.

---

## What NOT to Do

- Do not add file trees, grep commands, or framework explanations.
- Do not add "Definition of Done" checklists or code style rules enforced by linters.
- Do not duplicate information already in a parent AGENTS.md.
- Do not add aspirational rules — document what IS, not what SHOULD BE.
- Do not restructure a file that is working. Targeted edits only.

---

## Line Ceilings

These are smell tests, not targets. If a file exceeds the ceiling, it is documenting instead of routing — re-apply the primary filter and cut.

- Root AGENTS.md: under 120 lines
- Sub-folder AGENTS.md: under 90 lines

---

## Output

Show only the changed files. For each, provide:
- A one-line summary of what changed and why
- The diff or the updated file content
