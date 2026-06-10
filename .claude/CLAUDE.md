# .claude/CLAUDE.md — Claude Code Bootstrap

> **Source of truth: `.ai/AGENTS.md`** — read it first and follow it strictly.
> This file contains Claude Code–specific triggers only. It does not add or override rules.

---

## On every session start

1. Read `.ai/AGENTS.md` — it is the single source of truth for all rules.
2. Do not read root `AGENTS.md` — it is a pointer, not a rules file.

---

## Setup trigger

When human says any of:
`"setup ai workflow"` · `"setup workflow"` · `"init workflow"` · `"initialize workflow"`

→ Run `.ai/workflows/setup.md` — do not improvise a setup flow.

---

## Merge trigger (on setup, if existing files found)

If root `AGENTS.md` contains project-specific rules (auth pattern, constraints, build commands)
beyond the pointer text → extract and merge into `.ai/AGENTS.md` project-specific sections.

If `.claude/CLAUDE.md` contains project-specific rules beyond this bootstrap text
→ extract and merge into `.ai/AGENTS.md` project-specific sections.

After merge: overwrite both files with their pointer/bootstrap forms.

---

## Repo scan trigger

When human says `"run repo-scan"` or `"scan repo"`:
→ Run `.ai/workflows/repo-scan.md`