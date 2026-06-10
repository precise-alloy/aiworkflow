# Definition of Done — V1.0

## Hard gates — apply to all changes

- [ ] Project compiles with zero errors
- [ ] Lint passes with zero errors (warnings flagged, non-blocking)
- [ ] No new untyped values without inline comment explaining why (typed languages)
- [ ] No secrets, API keys, or credentials in changed files
- [ ] Scope clean: no files modified outside the task's stated CONTEXT

## Per change type

### New service method

- [ ] At least one unit test: happy path
- [ ] At least one unit test: primary failure path
- [ ] Error handling follows `code-conventions.md` pattern (no naked throws, no swallowed errors)

### New API endpoint

- [ ] Auth guard tested: 401 without valid token
- [ ] Tenant/ownership scoping tested: 403 for wrong entity
- [ ] Input validation present for required fields
- [ ] Structured error response returned to client (not stack trace)

### New background worker / job

- [ ] Dispatch logic tested
- [ ] Error logged on failure (job ID + error + stack)
- [ ] Retry behavior tested if applicable

### Database migration

- [ ] Migration file exists and runs cleanly
- [ ] Data integrity verified: row counts and schema shape spot-checked
- [ ] Irreversibility documented if migration cannot be rolled back
- [ ] Queries in migration use parameterized inputs

### Shared type / contract change

- [ ] All importing modules compile after the change
- [ ] Contract test passes (or documented why N/A with justification)

### UI change

- [ ] Visual testing checklist in `ui-visual-testing.md` completed
- [ ] Verified in browser at 375px and 1280px
- [ ] Accessible: keyboard nav, labels, contrast

### Safety override changes (auth, billing, migrations, tenant isolation, infra config, shared contracts)

- [ ] Full security checklist in `security.md` completed
- [ ] Audit log entry created for any state changes
- [ ] Classified STANDARD or above — no TRIVIAL/SIMPLE

## Soft gates — flag but non-blocking

- Lint warning count > 5 new: note in commit message
- Test omitted for specific edge case: document reason in DONE WHEN
- Visual test skipped: document reason — must be logic-only or config-only

## Quality thresholds

- No single function with cyclomatic complexity > 10 without a comment
- No function > 60 lines without structural justification
- No file > 400 lines — split at module boundary

## Completion signal

Report done only after:

1. All applicable hard gates above checked
2. All DONE WHEN items in task file verified
3. DOC UPDATE completed per doc trigger matrix in `generate-tasks.md`
4. Output block in `.ai/AGENTS.md` format produced