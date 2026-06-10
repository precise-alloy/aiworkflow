# AI Workflow — V1.0

A generic, project-agnostic AI-assisted development workflow. Covers role detection, triage, task generation, execution, standards enforcement, and doc discipline. Works with any language, stack, or team size.

---

## How to adopt

Two paths — pick one.

| Situation | Do this |
| --- | --- |
| New project (empty repo) | Copy `.ai/` folder into repo root → say `"setup ai workflow"` |
| Existing codebase | Copy `.ai/` folder into repo root → say `"setup ai workflow"` → then `"run repo-scan"` |

That's it. Claude runs an interactive setup interview — no templates to paste.

---

## What the setup interview asks

Claude asks one question at a time. Have these ready before starting:

| # | Question | Notes |
| --- | --- | --- |
| Q1 | New or existing project? | Determines whether repo-scan runs after setup |
| Q2 | Project name | Lowercase kebab-case — used in task file slugs and commit scopes |
| Q3 | Team size | `solo` / `small` / `medium` / `large` — drives tasks/ naming and memory expiry |
| Q4 | Primary language + framework | e.g. `TypeScript + Next.js`, `Python + FastAPI` |
| Q5 | Git platform | `GitHub` / `GitLab` / `Azure DevOps` / other — skipped if solo |
| Q6 | Git workflow | `PR-based` / `trunk-based` / `git-flow` |
| Q7 | AI tools in use | `Claude Code` / `Cursor` / `Codex` / `Cline` / other |
| Q8 | Model cost preference | `cost-first` / `balanced` (default) / `quality-first` |
| Q9 | Ticket format | e.g. `PROJ-123` / `#123` / `none` — skipped if solo |
| Q10 | Workflow owner | `tech-lead` / `team` — skipped if solo |
| Q11 | QA mode (optional) | `task` / `epic-only` / `off` (default) — generates `.qa.md` docs for testers |

---

## What gets generated

After the interview, Claude writes these files automatically:

| File | What it contains |
| --- | --- |
| `.ai/AGENTS.md` | **Source of truth** — all rules, team config, model routing, branch conventions |
| `.ai/exec-context.md` | Auto-generated executor context — thin subset for Codex/Cursor/Cline |
| `.ai/routing.md` | Role detection flow, context-by-role map, model routing table |
| `.ai/SKILLS-TODO.md` | Tech stack registry — language + framework pre-filled, rest as ❓ |
| `docs/ARCHITECTURE.md` | Header filled — rest populated progressively during work |
| `AGENTS.md` (root) | Thin pointer → `.ai/AGENTS.md` |
| `.claude/CLAUDE.md` | Bootstrap pointer + setup/repo-scan triggers |

**Filled later by repo-scan or first work session:**

- Auth pattern, error handling, testing pattern, project constraints
- Build / lint / test / typecheck commands
- Module skill files (`.ai/skills/`)
- Module map (`.ai/module-map.md`)

---

## Role system

Claude detects its role from the message before loading any context.

| Role | Triggered by | Context loaded | Output |
| --- | --- | --- | --- |
| **Architect** | `design`, `plan`, `break down`, `analyze`, `brainstorm`, `review and propose` | Full `.ai/AGENTS.md` | Task files + execution plan |
| **Executor** | `implement`, `fix`, `build`, `create`, `refactor`, `update X` | `exec-context.md` only (~75% fewer tokens) | Code changes + DONE WHEN |

Claude also acts as executor when the message is execution-intent — it loads `exec-context.md` instead of `AGENTS.md`. External tools (Codex, Cursor, Cline) always run as executor.

Ambiguous message → defaults to **Architect**. Claude never switches role mid-task silently — stops and asks if unclear.

---

## Daily workflow

```text
Write requirement
  → Claude detects role (architect or executor)

  ARCHITECT:
    → Triage (TRIVIAL / SIMPLE / STANDARD / EPIC)
    → TRIVIAL: Claude implements directly
    → SIMPLE:  2-section task note → send to executor tool
    → STANDARD/EPIC: task files + execution plan → send to executor tool

  EXECUTOR (Codex / Cursor / Cline):
    → Reads exec-context.md + task file + applicable standards
    → Implements STEPS
    → Validates DONE WHEN + standards gates
    → Updates .qa.md if QA mode is on
    → Reports done

  Human:
    → Reviews diff
    → Commits using batch commit message
    → Pushes
```

---

## QA docs (when QA mode is active)

For STANDARD and EPIC tasks, Claude generates a paired `.qa.md` alongside each task file.

```text
.ai/tasks/feat-auth/001-add-login.md        ← task (for executor)
.ai/tasks/feat-auth/001-add-login.qa.md     ← QA impact doc (for tester)
```

Each `.qa.md` contains: affected features, regression risk (low/medium/high), test scenarios, regression checks, and what testers can safely skip.

When an EPIC closes, a summary is generated at `docs/qa/{epic-slug}.qa.md` consolidating all impacted features in recommended test order.

---

## Repository structure

```text
AGENTS.md                    ← thin pointer to .ai/AGENTS.md (do not add rules here)
README.md                    ← this file
.claude/
  CLAUDE.md                  ← Claude Code bootstrap (setup trigger, pointer to .ai/AGENTS.md)
.ai/
  AGENTS.md                  ← SOURCE OF TRUTH — all rules, config, conventions
  exec-context.md            ← auto-generated executor context (Codex/Cursor/Cline)
  SKILLS-TODO.md             ← tech stack registry (❓ rows filled during work)
  module-map.md              ← phrase → module name mappings
  routing.md                 ← role detection, context-by-role, model routing
  workflows/
    setup.md                 ← setup interview (runs on "setup ai workflow")
    repo-scan.md             ← existing project scan (runs on "run repo-scan")
    generate-tasks.md        ← architect: triage, task formats, QA file formats
    execute-task.md          ← executor: implementation steps, DONE WHEN gates
    feature.md               ← feature workflow
    bugfix.md                ← bugfix workflow
  standards/
    code-conventions.md      ← naming, structure, error handling, logging
    security.md              ← auth, secrets, tenant isolation, audit logging
    testing-policy.md        ← test type matrix, unit/integration rules
    ui-visual-testing.md     ← component, page, accessibility checks
    definition-of-done.md    ← hard gates, per-change-type checklists
  skills/                    ← module interface summaries (created during work)
  tasks/                     ← execution task files (created during work)
  memory/                    ← multi-session EPIC context snapshots
docs/
  ARCHITECTURE.md            ← system architecture (filled progressively)
  CUTOFF.md                  ← module registry — documented vs code-only
  DECISIONS.md               ← architecture decisions and rationale
  CHANGELOG.md               ← completed EPICs log
  LESSONS.md                 ← dangerous patterns discovered from bugs
  qa/                        ← EPIC QA summaries (created when EPICs close)
```

---

## File merge strategy (existing projects)

| File / folder | On adopt |
| --- | --- |
| `.ai/AGENTS.md` | Source of truth — merge existing project constraints, keep filled values |
| `.ai/exec-context.md` | Regenerate from filled `.ai/AGENTS.md` after merge |
| `AGENTS.md` (root) | Overwrite with thin pointer after extracting any project-specific rules |
| `.claude/CLAUDE.md` | Overwrite with bootstrap form after extracting any project-specific rules |
| `.ai/workflows/*.md` | Replace |
| `.ai/routing.md` | Replace |
| `.ai/SKILLS-TODO.md` | Generate fresh from repo-scan |
| `.ai/skills/*.md` | Append only — never overwrite |
| `.ai/tasks/**` | Never touch |
| `.ai/memory/**` | Never touch |
| `docs/**` | Never touch |
| `.ai/module-map.md` | Never touch |

---

## Ongoing maintenance

### `.ai/AGENTS.md` — fill during work

| Section | When to fill |
| --- | --- |
| `{{PROJECT_CONSTRAINTS}}` | Add rules as discovered — one rule per line |
| `{{TESTING_PATTERN}}` | After first test — project-specific commands and rules |
| `{{AUTH_PATTERN}}` and sub-fields | During repo-scan or first auth-touching task |
| `{{ERROR_HANDLING_PATTERN}}` | During repo-scan |
| Build commands | During repo-scan or correct after setup if auto-detection was wrong |

### `docs/ARCHITECTURE.md` — fill progressively

| Section | When |
| --- | --- |
| Apps/services and purposes | During first EPIC or system overview session |
| Key flows | Each time a major flow is built |
| Runbook patterns | First time a pattern is used (new endpoint, new module, etc.) |

### `docs/DECISIONS.md` — one entry per non-obvious constraint

Write a `DECISION-NNN` block each time a rule is added to `.ai/AGENTS.md` that needs rationale.

### `docs/LESSONS.md` — one entry per dangerous pattern found

Two triggers: (1) after bugs that reveal "never again" patterns, (2) when an EPIC memory file expires — extract failure/lesson entries before deleting the memory file.

### `docs/CHANGELOG.md` — one entry per completed EPIC

One line per EPIC. Delete the corresponding `.ai/memory/` file after logging.

### `.ai/memory/` — EPIC context files expire

Expiry days set by team size (solo=60, small=30, medium=21, large=14). Before deleting an expired memory file:

1. Extract `## Decisions` → append to `docs/DECISIONS.md`
2. Extract lessons → append to `docs/LESSONS.md`
3. Write one-line summary → `docs/CHANGELOG.md`
4. Delete the memory file

### `docs/qa/` — EPIC QA summaries (when QA mode is active)

Generated automatically when an EPIC closes. One file per EPIC: `docs/qa/{epic-slug}.qa.md`. Testers use this as the primary reference for what to verify and in what order.

### `.ai/skills/{module}.md` — create on first touch

Written by Claude when a module's interface is first encountered. Max 150 lines. Updated when public interface changes.

---

## Key resources

| Resource | Purpose |
| --- | --- |
| [.ai/AGENTS.md](.ai/AGENTS.md) | Source of truth — rules, triage, model routing, auth, build commands |
| [.ai/routing.md](.ai/routing.md) | Role detection, context-by-role map, executor guide |
| [.ai/exec-context.md](.ai/exec-context.md) | Executor context (Codex/Cursor/Cline read this, not AGENTS.md) |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | System architecture, runbook patterns |
| [docs/CUTOFF.md](docs/CUTOFF.md) | What is documented vs what exists only in code |
| [.ai/standards/](.ai/standards/) | Code conventions, security, testing, UI, definition of done |
| [docs/qa/](docs/qa/) | EPIC QA summaries for testers |