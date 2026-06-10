# AI Workflow — V1.0

Developer guide for this project's AI-assisted development workflow.

## Structure

```text
.github/agents/ai-workflow.md          ← rules for all AI agents (repo root — auto-loaded)
.ai/
  SKILLS-TODO.md   ← tech stack registry, fill during work
  module-map.md    ← module name mappings
  routing.md       ← flow, roles, executor guide
  workflows/       ← agent runbooks (generate-tasks, execute-task, feature, bugfix)
  standards/       ← code conventions, security, testing, UI, definition of done
  skills/          ← module interface summaries (created during work)
  tasks/           ← execution task files, grouped by module (created during work)
  memory/          ← multi-session context snapshots for EPICs (created during work)
docs/
  CUTOFF.md        ← module registry (what's documented vs not)
  ARCHITECTURE.md  ← system architecture
  DECISIONS.md     ← key architecture decisions + rationale
  CHANGELOG.md     ← completed EPICs log
```

---

## Setup

**New project:**

```text
Tell Claude: "Setup AI workflow for new project"
→ Claude generates skeletons, SKILLS-TODO.md all ❓
→ Begin work — Claude fills stack info progressively
```

**Existing project:**

```text
Tell Claude: "Setup AI workflow for existing project"
→ Claude runs repo-scan → conflict check → you confirm
→ Claude merges/replaces files, fills SKILLS-TODO from scan
→ Begin work
```

See root `README.md` for copy-paste setup templates with all required fields.

---

## Daily flow

```text
1. Write requirement
   → Claude triages (TRIVIAL / SIMPLE / STANDARD / EPIC)

   TRIVIAL:         Claude implements directly — no task file
   SIMPLE:          Claude writes 2-section task note → paste into Codex
   STANDARD / EPIC: Claude generates task files + execution plan
                    → Paste Codex group → Codex panel
                    → Paste Cline group → Cline terminal (if needed)

2. Review diff → commit using batch commit message → push
```

---

## Requirement template

```text
MODULE: {name — see .ai/module-map.md}
TYPE: feature | bugfix | doc-debt | migration | epic
GOAL: {1–2 sentences}
ACCEPTANCE CRITERIA:
  - {condition 1}
  - {condition 2}
SCOPE CONSTRAINTS: {optional}
RELATED FILES: {optional}
```

---

## Executor guide

| Triage level    | Action                                                           |
|-----------------|------------------------------------------------------------------|
| TRIVIAL         | Claude implements directly — no paste needed                     |
| SIMPLE          | Paste 2-section task note into Codex                             |
| STANDARD / EPIC | Paste full task file into Codex (code edits), then Cline (shell) |

Codex quota exhausted: paste same task file into Cline instead.
Codex groups always first. Cline (shell) tasks always last.

---

## Commit format

```text
type(scope): subject

- what changed and why
- key invariant enforced

Breaking: none | {what breaks}
Migration: none | {migration needed}
```

Types: `feat` | `fix` | `refactor` | `test` | `docs` | `chore`

---

## File merge strategy (when adopting into existing project)

| File/folder                     | On adopt                         |
|---------------------------------|----------------------------------|
| `.github/agents/ai-workflow.md` | Merge — keep project constraints |
| `.ai/workflows/*.md`            | Replace                          |
| `.ai/routing.md`                | Replace                          |
| `.ai/SKILLS-TODO.md`            | Generate fresh                   |
| `.ai/skills/*.md`               | Append only — never overwrite    |
| `.ai/tasks/**`                  | Never touch                      |
| `.ai/memory/**`                 | Never touch                      |
| `docs/**`                       | Never touch                      |
| `.ai/module-map.md`             | Never touch                      |

---

## When Claude hits a ❓ in SKILLS-TODO.md

Claude stops, asks once, fills the row, updates `.github/agents/ai-workflow.md   `, continues.

---

## Memory files

Check header before Claude uses one:

```text
Auto-expire: YYYY-MM-DD
Active-until: {task ID} done
```

When EPIC done: delete file + log in `docs/CHANGELOG.md`.

---

## Optional scripts

```bash
pnpm validate-all     # batch EPIC sanity check
pnpm docs-sync        # pre-PR doc check
pnpm lint-workflows   # workflow drift check
```