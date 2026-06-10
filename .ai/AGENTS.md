# AGENTS.md — AI Workflow V1.0

> **Source of truth.** Root `AGENTS.md` and `.claude/CLAUDE.md` point here.
> All rules, conventions, and project config live in this file only.

Project: `{{PROJECT_NAME}}`
Stack: `{{PRIMARY_LANGUAGE}} + {{FRAMEWORK}}`

## Language

Primary: `{{PRIMARY_LANGUAGE}}`
Docs and comments: English only.

---

## Team config

<!-- Filled by setup wizard — do not edit manually -->

| Field | Value |
| --- | --- |
| Team size | `{{TEAM_SIZE}}` |
| Git platform | `{{GIT_PLATFORM}}` |
| Git workflow | `{{GIT_FLOW}}` |
| AI tools in use | `{{AI_TOOLS}}` |
| Workflow owner | `{{WORKFLOW_OWNER}}` |
| Ticket format | `{{TICKET_FORMAT}}` |
| EPIC memory expiry | `{{MEMORY_EXPIRY_DAYS}}` days |
| QA mode | `{{QA_MODE}}` |

---

## Role detection (run before triage — before any further context load)

Detect role from message intent. Load the rest of this file only if ARCHITECT.

**ARCHITECT** — message signals: `design` · `plan` · `architect` · `break down` · `generate tasks` · `review and propose` · `brainstorm` · `analyze` · `how should we` · `what's the approach` · `what should we`

→ Continue loading this file. Run triage. Generate task files or fix directly per triage level.

**EXECUTOR** — message signals: `implement` · `fix` · `build` · `add` · `create` · `write` · `refactor` · `update <specific thing>` · `change <specific thing>`

→ Stop loading this file. Load `.ai/exec-context.md` instead. Implement task directly.

**Default:** ambiguous message → **ARCHITECT**. Plan before acting.

**Mid-task rule:** if role becomes unclear during work — stop and ask. Never switch context mid-task silently.

---

## Triage (mandatory — before any context load)

Classify every incoming change before loading context or generating tasks.
See `.ai/workflows/generate-tasks.md` TRIAGE section for full rules.

| Level | Criteria | Output |
| --- | --- | --- |
| TRIVIAL | Single file, no public contract change, clear validation, low blast radius | Direct fix — no task file |
| SIMPLE | ≤2 files, no cross-module contract, clear path, low blast radius | Task note: TASK + DONE WHEN only |
| STANDARD | Cross-file, design decision, public contract changed, or validation unclear | Full task file |
| EPIC | Multi-session, multiple modules, new architecture pattern | Full task file + memory file |

**Safety override — force STANDARD regardless of above when touching:**
auth / session / tokens · payment / billing · database migrations · tenant isolation · infra / runtime config · shared contracts (types exported across modules)

---

## Pre-coding read order (load per classification — not always all steps)

**TRIVIAL:** `.ai/AGENTS.md` golden rules + grep target file only.

**SIMPLE:** `.ai/AGENTS.md` + `.ai/skills/<module>.md` only.
Stale check: grep-verify 1–2 key names the skill file claims exist. If missing: flag ⚠️ before trusting.

**STANDARD / EPIC:**

1. `.ai/AGENTS.md`
2. `docs/LESSONS.md` — grep module/file name; load matching entries only. **Load before skill files** — known failure patterns must inform how you read them. If no match: skip.
3. `docs/CUTOFF.md`
4. `.ai/SKILLS-TODO.md` — check ❓ rows before starting; if ❓ found: stop, ask human once, fill, update this file, continue
5. `.ai/skills/{module}.md` — **stale check required**: grep-verify 2+ key claims (function names, exports, types) exist in source before trusting. If any missing: flag ⚠️ stale, note which claims are outdated, re-read source for those only.
6. `docs/modules/{module}/` — only if in CUTOFF.md AND no skill file
7. `docs/ARCHITECTURE.md` — only if task involves new resource/endpoint/module
8. `.ai/memory/{branch-slug}-{feature-slug}-context.md` — only if resuming multi-session

**Token cap:** when task touches > 3 modules, load skill files only for directly modified modules. For others: grep entry point only — do not load full skill file.

**Skill file refresh trigger:** if stale check fails on 3+ claims, rewrite the skill file from source before proceeding. Do not patch a partially-wrong skill file — rewrite it clean.

Do not scan the entire repo.

---

## Task-first rule

Check `.ai/tasks/` for a matching task before creating new work.
If found: execute or update. Do NOT create duplicates.

---

## Golden rules

1. Triage before context load — classify first, load only what the level requires.
2. Minimal change — no unsolicited refactors.
3. Grep before edit — confirm paths exist before touching files.
4. No hallucinated features — if unclear, stop and ask.
5. One final report — no intermediate dumps.
6. Stop on errors (compile fail, test fail, 4xx/5xx).
7. Done = DONE WHEN conditions met AND applicable standards hard gates pass. Both are required.
8. Update docs only when the change affects how future humans or agents understand, navigate, or safely modify the system — use the doc trigger matrix in `generate-tasks.md`.
9. Task STEPS use positive instructions only.
10. Grep before claiming — before asserting any fact about existing codebase behavior, grep or read the source. If unverifiable in current context, say so explicitly. Never infer. Cite `file:line` when making a claim about code.

---

## Skills/ maintenance

After any task that changes a module's public interface:

- Check if `.ai/skills/{module}.md` exists
- If yes: add update to DOC UPDATE section of the task
- If no: create a stub skill file after the task completes

Skill files stay under 150 lines. Interface summary only — not a replacement for ARCHITECTURE.md.

---

## SKILLS-TODO.md discipline

When a ❓ row is encountered mid-task:

1. Stop
2. Ask human once: "What is `<role>` for this project?"
3. Human answers → fill the row → mark ✅
4. Update `.ai/AGENTS.md` relevant section
5. Continue task

Never guess a ❓ value. Never ask more than one question at a time.

---

## Model routing

<!-- Filled by setup wizard based on model cost preference -->
<!-- Replace this block with the correct table from .ai/workflows/setup.md §Derived values -->

`{{MODEL_ROUTING}}`

---

## Branch & PR conventions

<!-- Filled by setup wizard -->

### Branch naming

```text
{{BRANCH_FORMAT}}
```

### tasks/ path convention

```text
{{TASKS_PATH_CONVENTION}}
```

When creating a task file, the branch slug = current git branch name with `/` replaced by `-`.

### PR requirements

- Title must match commit convention: `type(scope): subject`
- Body must include: what changed, why, DONE WHEN conditions verified
- Required reviewers: `{{PR_REVIEWERS}}`
- Link ticket if `{{TICKET_FORMAT}}` is not `none`
- No PR merges with failing CI

### DONE WHEN gate (added for all STANDARD / EPIC tasks)

```text
[ ] No claim made about existing code without citing file:line
```

---

## Prompt caching strategy

Split every API call into **stable** (cached) and **variable** (uncached) segments.

**Stable — send as system prompt with `cache_control: ephemeral`:**

- Full `.ai/AGENTS.md` content
- Loaded `standards/*.md` files
- Loaded `skills/{module}.md` files

**Variable — send in user message (never cache):**

- Task file content
- Per-task context: diff, file excerpts, research snapshot
- Any content that changes between requests

```text
system:
  [.ai/AGENTS.md full text]    ← cache_control: ephemeral
  [standards/relevant.md]      ← cache_control: ephemeral
  [skills/module.md if loaded] ← cache_control: ephemeral

user:
  [task file]
  [file excerpts grep'd for this task]
```

**Rule:** Never mix stable and variable content in the same prompt block. Variable content in the system prompt breaks cache reuse.

---

## Standards

Load relevant standards before implementation. Match to task type — not all for every task.

| Task touches | Load |
| --- | --- |
| Any code change | `.ai/standards/code-conventions.md` |
| Auth, billing, migrations, tenant isolation, infra config, shared contracts | `.ai/standards/security.md` |
| New service / endpoint / worker / bug fix | `.ai/standards/testing-policy.md` |
| Frontend component or page | `.ai/standards/ui-visual-testing.md` |
| Any STANDARD or EPIC task | `.ai/standards/definition-of-done.md` |

Validate against loaded standards before reporting done. Standards are part of DONE WHEN.

---

## Authentication

<!-- Filled by repo-scan — see .ai/workflows/repo-scan.md -->

Auth pattern: `{{AUTH_PATTERN}}`

- Token verification: `{{AUTH_TOKEN_VERIFICATION}}`
- Tenant/workspace scoping: `{{AUTH_TENANT_SCOPING}}`
- Role/permission model: `{{AUTH_ROLE_MODEL}}`

---

## Error handling

<!-- Filled by repo-scan — see .ai/workflows/repo-scan.md -->

`{{ERROR_HANDLING_PATTERN}}`

---

## Testing

Full test policy: see `.ai/standards/testing-policy.md`

<!-- Filled by repo-scan — project-specific commands and key rules -->

`{{TESTING_PATTERN}}`

---

## Project-specific constraints

<!-- Fill during work — rules specific to this project not covered by golden rules or standards above -->

`{{PROJECT_CONSTRAINTS}}`

---

## Build commands

<!-- Filled by repo-scan — see .ai/workflows/repo-scan.md -->

| Command | What |
| --- | --- |
| `{{BUILD_CMD}}` | Build project |
| `{{TYPECHECK_CMD}}` | Type check only |
| `{{TEST_CMD}}` | Run tests |
| `{{LINT_CMD}}` | Lint |

---

## Output format (mandatory)

```text
Files changed:
- path — summary

Docs updated:
- path — what changed   (or: none required)

Commit message:
type(scope): subject

- what changed and why
- key invariant enforced (if any)

Breaking: none | <what breaks>
Migration: none | <migration needed>

DONE WHEN verified: ✓ all conditions met
```

Commit types: `feat` | `fix` | `refactor` | `test` | `docs` | `chore`