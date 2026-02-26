# Task: Generate SKILL.md Files for Factory Droid

## What is a Skill?

A Factory Droid skill is a focused package of **procedural knowledge** -- step-by-step instructions for completing a specific type of task in your codebase. Skills live in `.factory/skills/<skill-name>/SKILL.md` and are automatically invoked by the Droid when relevant, or manually via `/skill-name`.

## Research-Backed Principles

Based on empirical research (SkillsBench, Li et al. 2026):

1. **Focused skills with 2-3 sections outperform comprehensive documentation** -- keep each skill under 40 lines of instruction content.
2. **Self-generated skills provide zero average benefit** -- these must be human-reviewed and curated to be useful.
3. **Only encode procedures agents can't infer** -- agents already know how to run tests, read types, and navigate file trees. Don't restate that.
4. **SWE tasks get the smallest boost** (+4.5pp) from skills -- so every line must earn its place. Generic coding knowledge adds nothing.

## What Belongs in a Skill vs. AGENTS.md

| SKILL.md (procedural "how to")                | AGENTS.md (declarative "what is")         |
|------------------------------------------------|-------------------------------------------|
| Step-by-step procedure for a specific task     | Repository-wide conventions               |
| Which existing file to copy as a template      | Import rules, naming conventions          |
| Non-obvious gotchas for THIS workflow          | Security guardrails                       |
| When to use / when NOT to use this skill       | Directory structure pointers              |

**Rule**: If the information applies to ALL tasks in the repo, it belongs in AGENTS.md. If it applies only to a specific type of task, it belongs in a skill.

---

## Phase 1: Identify Skills Worth Creating

For each candidate skill, apply this filter:

1. **Is this task repo-specific?** Generic tasks (run tests, lint code, create a file) don't need skills.
2. **Does the procedure have non-obvious steps?** If an agent would figure it out by reading 1-2 files, skip it.
3. **Has an agent (or developer) gotten this wrong before?** If yes, the skill encodes the correction.

Common high-value skills:
- Tasks involving multiple coordinated files (e.g., adding a provider requires file + registry + config)
- Tasks with repo-specific patterns that differ from framework defaults
- Tasks touching dangerous/complex areas (large files, auth, billing)

Common low-value skills (skip these):
- "Run the test suite" -- agents know how
- "Create a React component" -- standard framework knowledge
- "Fix lint errors" -- agents handle this natively

---

## Phase 2: Generate Each SKILL.md

**Target: 30-50 lines of instruction content** (excluding frontmatter).

### File Format

```markdown
---
name: skill-name
description: [One sentence: what it does + when to use it. The Droid uses this to decide when to invoke the skill.]
---
# Skill: [Display Name]

## When to use

- [2-4 bullet points: specific triggers for this skill]

## Critical conventions (non-obvious)

- [Only rules an agent would violate without being told]
- [Repo-specific patterns that differ from framework defaults]
- [Known gotchas that have caused real bugs]

## Steps

1. [Pick the closest existing file as template: `path/to/example.ts`]
2. [Concrete procedural step with file path]
3. [Next step...]
4. Run verification: `[single command]`

## Safety

- [When to stop and ask for human review]
- [Dangerous areas to avoid or handle carefully]
```

### Section Rules

**Frontmatter** (required):
- `name`: lowercase, hyphenated. This becomes the `/slash-command`.
- `description`: one sentence covering WHAT and WHEN. This is the most important line -- the Droid reads it to decide whether to invoke the skill.

**When to use** (required, 2-4 bullets):
- Specific, concrete triggers. Not "when you need to change something."
- Include "out of scope" only if the boundary is genuinely confusing (e.g., Express vs. Remix routes).

**Critical conventions** (required, 3-8 bullets):
- ONLY non-obvious rules. Apply the test: "Would an agent get this right 80% of the time without being told?" If yes, cut it.
- Never restate AGENTS.md content. Reference it: "See AGENTS.md for import rules."
- Concrete examples beat abstract rules.

**Steps** (required, 3-8 steps):
- Always start with "pick the closest existing file as template" and name 2-4 real files.
- Each step references a real file path.
- End with the verification command.

**Safety** (required, 1-3 bullets):
- When to escalate to human review.
- Dangerous files or areas (with reason).

### What NOT to Include

- **Key files tables** listing obvious files (tsconfig.json, package.json, etc.)
- **Type definitions** -- agents read the actual source
- **grep/find commands** -- agents know how to search
- **Framework explanations** -- agents know React, Remix, Express
- **Full code templates** -- one short example is enough; agents adapt from real files
- **Inputs section** -- if the user describes the task, the agent has the inputs
- **Completion criteria** that just restate "verification passes"

---

## Phase 3: Decide Invocation Mode

For each skill, set frontmatter flags:

| Skill type | Frontmatter | Rationale |
|------------|-------------|-----------|
| Standard workflow | (defaults) | Both user and Droid can invoke |
| Dangerous workflow (deploy, migrate) | `disable-model-invocation: true` | User must explicitly trigger |
| Background knowledge (legacy context) | `user-invocable: false` | Droid uses when relevant, not a command |

---

## Output Format

```
---
File: `.factory/skills/<name>/SKILL.md`
---
[content]

[...repeat for each skill...]
```

---

## Final Checklist

### Information Diet
- [ ] Each skill is under 50 lines of instruction content
- [ ] Every bullet passes: "Would an agent get this wrong without being told?"
- [ ] No duplication with AGENTS.md
- [ ] No framework explanations, grep commands, or type definitions
- [ ] No "Key files" tables of obvious files

### Procedural Quality
- [ ] Steps reference real, existing files as templates
- [ ] Steps are ordered (what to do first, second, third)
- [ ] Verification command is a single copy-paste line
- [ ] Safety section names specific dangerous areas with reasons

### Skill Selection
- [ ] Each skill represents a genuinely non-trivial, repo-specific workflow
- [ ] No skills for generic tasks agents handle natively
- [ ] Description frontmatter clearly tells the Droid WHEN to invoke
