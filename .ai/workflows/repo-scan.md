# repo-scan.md — V1.0

## Trigger

Run this workflow when:

- Human says `"run repo-scan"` or `"scan repo"`
- Setup wizard completes with `{{PROJECT_TYPE}} = existing`
- A new developer adopts this workflow on an existing codebase

---

## Purpose

Populate all `[fill after repo-scan]` placeholders in `.ai/profiles/project.md` and `SKILLS-TODO.md`
without reading the entire codebase. Targeted detection only.

---

## Behavior rules

- Grep and read entry points only — do not scan the full repo
- Fill what you can detect with confidence; mark `❓` for anything ambiguous
- Never guess — if detection is unclear, leave `❓` and note what to ask human
- After filling, output a summary report and a list of remaining `❓` rows

---

## Step 1 — detect stack (fills SKILLS-TODO.md core rows)

Run these checks in parallel:

| Check               | Command                                                                                  | Fills               |
|---------------------|------------------------------------------------------------------------------------------|---------------------|
| Language            | Look for `package.json`, `pyproject.toml`, `*.csproj`, `go.mod`, `Cargo.toml`, `pom.xml` | Primary language    |
| Runtime / framework | Read deps from detected manifest                                                         | Runtime / framework |
| Package manager     | Check for `pnpm-lock.yaml`, `yarn.lock`, `package-lock.json`, `poetry.lock`, `uv.lock`   | Package manager     |
| Build tool          | Check `scripts` in package.json or `Makefile`, `justfile`, `build.gradle`                | Build tool          |
| Test framework      | Grep deps for `jest`, `vitest`, `pytest`, `go test`, `rspec`, `junit`                    | Test framework      |
| Frontend framework  | Grep deps for `react`, `vue`, `svelte`, `angular`, `next`, `nuxt`                        | Frontend framework  |
| CSS approach        | Grep deps for `tailwind`, `styled-components`, `emotion`, `sass`, `css-modules`          | CSS approach        |
| Database            | Grep deps or config for `pg`, `mysql2`, `sqlite3`, `mongodb`, `prisma`, `drizzle`        | Database            |
| ORM / query builder | Grep deps for `prisma`, `drizzle`, `typeorm`, `sequelize`, `sqlalchemy`, `knex`          | ORM / query builder |
| Deployment target   | Look for `Dockerfile`, `fly.toml`, `vercel.json`, `railway.toml`, `.github/workflows/`   | Deployment target   |

Mark each found row `✅`. Mark each not found `❓` with a note of what to ask.

---

## Step 2 — detect auth pattern (fills `.ai/profiles/project.md` Authentication section)

```text
1. Grep for: middleware, auth, jwt, session, cookie, bearer, token, passport, nextauth, clerk, supabase
2. Read the top 2–3 matching files
3. Identify:
   - Auth mechanism (JWT / session cookie / API key / OAuth / third-party)
   - Where verification happens (middleware vs per-handler)
   - How tenant/workspace scoping is passed (session claim vs request body vs header)
   - Role/permission model if any
4. Fill {{AUTH_PATTERN}}, {{AUTH_TOKEN_VERIFICATION}}, {{AUTH_TENANT_SCOPING}}, {{AUTH_ROLE_MODEL}} in `.ai/profiles/project.md`
5. If ambiguous: fill what is clear, mark rest ❓
```

---

## Step 3 — detect error handling pattern (fills `.ai/profiles/project.md` Error handling section)

```text
1. Grep for: catch, error, throw, HttpException, APIError, logger, log.error
2. Read 1–2 representative files (API handler + background job if both exist)
3. Identify:
   - API error response shape
   - Logging approach (structured vs console)
   - Background job failure behavior
4. Fill {{ERROR_HANDLING_PATTERN}} in `.ai/profiles/project.md` with a concise 2–3 line description
5. If inconsistent across the codebase: note the inconsistency, pick the dominant pattern
```

---

## Step 4 — detect test setup (fills `.ai/profiles/project.md` Testing section + build commands)

```text
1. Read package.json scripts (or equivalent in detected build tool)
2. Extract: build, typecheck, test, lint commands
3. Fill {{BUILD_CMD}}, {{TYPECHECK_CMD}}, {{TEST_CMD}}, {{LINT_CMD}} in `.ai/profiles/project.md`
4. Look for test config files: jest.config.*, vitest.config.*, pytest.ini, etc.
5. Identify test conventions: unit vs integration separation, mock strategy, DB usage
6. Fill {{TESTING_PATTERN}} and {{TEST_LOCATION_CONVENTIONS}} in `.ai/profiles/project.md` with the key rules (2–4 bullet points max)
```

---

## Step 5 — generate exec-context.md (auto-generated from `.ai/profiles/project.md`)

After Steps 2–4 have filled auth, error handling, and build commands into `.ai/profiles/project.md`:

```text
1. Open .ai/exec-context.md template
2. Copy from filled `.ai/profiles/project.md` into exec-context.md:
   - {{AUTH_PATTERN}}, {{AUTH_TOKEN_VERIFICATION}}, {{AUTH_TENANT_SCOPING}}, {{AUTH_ROLE_MODEL}}
   - {{ERROR_HANDLING_PATTERN}}
   - {{BUILD_CMD}}, {{TYPECHECK_CMD}}, {{TEST_CMD}}, {{LINT_CMD}}
   - {{PROJECT_NAME}}
3. Do NOT copy: triage, team config, model routing, branch conventions,
   prompt caching, pre-coding read order, SKILLS-TODO.md discipline, memory expiry
4. Static sections stay as written in the template — do not overwrite from AGENTS.md
5. Write the filled exec-context.md
```

---

## Step 6 — build module map (fills .ai/module-map.md)

```text
1. List top-level directories in src/ (or equivalent source root)
2. For each directory: infer its domain from name + a quick grep of its entry file
3. Write entries to .ai/module-map.md:
   <human phrase> -> <kebab-case-name>
4. Add catch-all: default -> misc
5. Do not invent modules — only map what exists
```

---

## Step 7 — populate CUTOFF.md skeleton (fills docs/CUTOFF.md)

```text
1. For each module identified in Step 5:
   - Add a row to the "Undocumented" table in CUTOFF.md
   - Status: "source only — read directly"
2. Leave "Documented" table empty — filled as skill files are created during work
```

---

## Step 8 — pre-flight conflict check (existing projects only)

Before merging any workflow files, check:

```text
1. Does .ai/AGENTS.md already exist?
   → If yes: diff new template against existing — list conflicts in report
2. Do .ai/workflows/*.md files already exist?
   → If yes: note which will be replaced (per routing.md merge strategy)
3. Are there existing .ai/tasks/ or .ai/memory/ files?
   → If yes: never touch them — list them as "preserved"
4. Does docs/ contain existing ARCHITECTURE.md, DECISIONS.md, LESSONS.md?
   → If yes: never touch them — list as "preserved"
```

Output conflict report. **Wait for human confirmation before writing any files.**

---

## Step 9 — apply merge strategy (after human confirms)

Apply per `routing.md` merge strategy:

| File/folder           | Action                                                                                   |
|-----------------------|------------------------------------------------------------------------------------------|
| `.ai/AGENTS.md`       | Merge workflow rule changes only — project constraints move to `.ai/profiles/project.md` |
| `.ai/profiles/*.md`   | Merge filled project/team/runtime values — preserve existing project constraints         |
| `.ai/exec-context.md` | Regenerate from filled AGENTS.md (Step 5)                                                |
| `.ai/workflows/*.md`  | Replace with new versions                                                                |
| `.ai/routing.md`      | Replace                                                                                  |
| `.ai/SKILLS-TODO.md`  | Generate fresh from Step 1 results                                                       |
| `.ai/skills/*.md`     | Append only — never overwrite                                                            |
| `.ai/tasks/**`        | Preserve — never touch                                                                   |
| `.ai/memory/**`       | Preserve — never touch                                                                   |
| `docs/**`             | Preserve — never touch                                                                   |
| `.ai/module-map.md`   | Write if not exists; append if exists                                                    |

---

## Output — scan report

After completing all steps, output:

```text
## Repo scan complete — {{PROJECT_NAME}}

### Stack detected (SKILLS-TODO.md updated)
✅ Language: <value>
✅ Framework: <value>
✅ Package manager: <value>
... (all ✅ rows)

### Needs human input (❓ rows)
❓ <role>: <what was ambiguous and why>
...

### .ai/profiles/project.md filled
✅ Auth pattern: <summary>
✅ Error handling: <summary>
✅ Build commands: <list>
✅ Test pattern: <summary>

### .ai/exec-context.md generated
✅ Auth values propagated from project profile
✅ Error handling propagated from project profile
✅ Build commands propagated from AGENTS.md

### Files preserved (not touched)
- .ai/tasks/** — N task files
- .ai/memory/** — N memory files
- docs/** — N docs files

### Conflicts found (requires human decision)
- <file>: <conflict description>
... (or: none)

Next step: resolve ❓ rows, then begin work.
```

---

## Hard rules

- Never write files before outputting the conflict report and receiving human confirmation
- Never touch `.ai/tasks/`, `.ai/memory/`, `docs/` unless explicitly told to
- Never guess `❓` values — ask human once per unknown
- Scan is read-only until Step 8
