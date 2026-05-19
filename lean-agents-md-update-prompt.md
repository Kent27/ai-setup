# Task: Update Existing AGENTS.md Files

## Rules

- Read each existing AGENTS.md before touching it.
- Make targeted edits only — do not rewrite or restructure unless the current structure is broken.
- Preserve all high-signal content. Remove only what is outdated, duplicated, or obviously wrong.
- Add only what an agent would get wrong without being told. If the code makes it obvious, omit it.
- Keep files under 150 lines. Shorter is better.

## What to Update

1. **Stale references** — file names, paths, package versions that no longer exist.
2. **Missing gotchas** — new patterns introduced since the file was last updated that have caused or would cause agent mistakes.
3. **Outdated conventions** — rules that no longer reflect how the codebase actually works.
4. **JIT Index gaps** — new sub-directories with non-obvious patterns that need their own AGENTS.md and a pointer from root.

## What NOT to Do

- Do not add file trees, directory descriptions, or framework explanations.
- Do not add "Definition of Done" checklists.
- Do not duplicate information already in a parent AGENTS.md.
- Do not add aspirational rules — only document what IS, not what SHOULD BE.

## Output

Show only the changed files with a one-line summary of what changed and why.
