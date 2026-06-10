---
name: pr-review
description: 'Review Azure DevOps pull requests or local pre-PR changes against requirements in Jira tickets. USE FOR: reviewing PRs, reviewing local branch/worktree changes before opening a PR, checking code changes against acceptance criteria, identifying potential issues, suggesting fixes and unit tests. DO NOT USE FOR: creating PRs, merging PRs, or deploying code.'
argument-hint: 'Provide either the Azure DevOps PR URL or describe the local branch/worktree to review, plus relevant Jira ticket URL(s)'
---

# Pull Request Review

Review Azure DevOps pull requests or local pre-PR changes against Jira ticket requirements, identify issues, suggest fixes, and provide unit tests.

## ⛔ MANDATORY: Remote API access

**Every Jira and Azure DevOps REST call in this skill MUST go through `scripts/remote-api.ts` via `bun run`.** No exceptions. Do not use `Invoke-WebRequest`, `curl`, `fetch`, `requests`, the `mcp_gitkraken_cli_*` or `mcp_github_mcp_se_*` tools, or any hand-rolled auth header. The helper loads `.env.local`, applies the correct Basic auth format for each system, and preserves HTTP status on failure.

If the command you need is not listed, run `bun run scripts/remote-api.ts` with no arguments to print the current subcommand list:

```bash
bun run scripts/remote-api.ts                                        # print usage
bun run scripts/remote-api.ts check-env                              # verify .env.local + required keys
bun run scripts/remote-api.ts jira issue <KEY_OR_URL> [fields]       # single ticket
bun run scripts/remote-api.ts jira comments <KEY_OR_URL>             # all comments
bun run scripts/remote-api.ts jira changelog <KEY_OR_URL>            # change history
bun run scripts/remote-api.ts ado pr <PR_URL>                        # PR metadata
```

If a needed call is missing from the helper, **stop and ask the user before falling back to anything else.** Extending the helper is preferred over working around it.

## AI Workflow Integration

This skill operates as **ARCHITECT** role within the `.ai/` workflow.

### Before starting:

1. Read `.ai/AGENTS.md` — follow golden rules.
2. **Always start from Step 1** (Gather Input) — ask the user whether this is a PR review or a local pre-PR review, then gather the relevant URLs/context. Never silently load existing review/task files or resume previous sessions.
3. After gathering input, load `.ai/standards/definition-of-done.md` — verify all hard gates pass.
4. Grep `docs/LESSONS.md` for the affected module/file names — flag if known failure patterns are being repeated.
5. Load `.ai/standards/stakeholders.md` — know who authored/owns what and who to escalate questions to.
6. Check `.ai/tasks/` for task files matching the reviewed branch — verify DONE WHEN conditions are met.

### Review checklist (in addition to §2.5 below):

- [ ] All DONE WHEN conditions in the task file are satisfied
- [ ] No claim about existing code without `file:line` citation
- [ ] Skill files updated if public interface changed (per `.ai/AGENTS.md` Skills/ maintenance)
- [ ] `.ai/standards/testing-policy.md` — required test types present
- [ ] `.ai/standards/security.md` — gates pass (if auth/billing/tenant touched)
- [ ] QA file (`.qa.md`) verified by executor (if QA mode is `task`)
- [ ] Adversarial pass completed — at least one fault-injection scenario attempted per changed method
- [ ] Every acceptance criterion mapped to a specific `file:line` or marked missing
- [ ] Blast radius analysis — all upstream callers of changed interfaces verified
- [ ] **Code comments with ticket ID** — verify all changed code includes comments explaining WHY + relevant ticket ID (e.g., `// FMI-XXX: reason`). Flag any uncommented changes.
- [ ] **Documentation up-to-date** — verify affected docs (skill files, ARCHITECTURE.md, DECISIONS.md, LESSONS.md) were updated to reflect the PR changes. Flag stale or missing doc updates.

## Prerequisite

Ensure the user has a `.env.local` file at the workspace root with the following keys populated:

```dotenv
JIRA_PAT=
JIRA_EMAIL=
AZURE_DEVOPS_READONLY_PAT=
AZURE_DEVOPS_EMAILS=
```

These PATs are the required source of truth for remote data access. The remote-api helper fetches Azure DevOps PR metadata and Jira ticket data using PAT-backed REST API calls before relying on local diffs or prior notes.

**Before proceeding, always run the sanctioned env check via Bun. Do not use `Test-Path`, `Get-Content`, `cat .env.local`, or any other ad-hoc check — results differ across shells and lead to false positives.**

```bash
bun run scripts/remote-api.ts check-env
```

Interpret the result:

- **Exit 0** (`ok: true`) — file exists and all four keys are present and non-empty. Proceed.
- **Exit 1 with `envLocalExists: false`** — create `.env.local` at the workspace root with the four keys shown above (empty values are fine), then tell the user to fill them in and re-run the skill.
- **Exit 1 with `missing: [...]`** — tell the user which keys are missing/empty and ask them to fill those values, then re-run the skill.

When using either PAT during the skill, treat any authentication or authorization failure (`401`, `403`, invalid credentials, PAT expired/revoked) as a hard stop for remote review. Ask the user to provide a valid PAT for the failing system, then retry the remote fetch before continuing.

## Step 1: Gather Input

Ask the user for:

1. **Review mode**:
    - PR review: the Azure DevOps pull request URL (e.g., `https://episerveremea-expertservices.visualstudio.com/First%20Mile/_git/FirstMile/pullrequest/8577`)
    - Local pre-PR review: the branch to review and whether the scope is committed branch diff, staged changes, unstaged changes, or the full worktree
2. The relevant Jira ticket URL(s) (e.g., `https://episerver-services.atlassian.net/browse/FMI-934`)
3. Any additional information or context that is not captured in the ticket (e.g., verbal decisions, Slack discussions, architectural constraints, priority notes)
4. The intended base branch if it is not `develop`

## Step 2: Review the Pull Request

Choose the review path based on the input mode.

### 2.1 Identify Review Target

#### PR review mode

Fetch remote PR details **only** via `scripts/remote-api.ts` (see ⛔ MANDATORY block at top of this file). Do not hit `dev.azure.com` directly and do not use any other MCP/GitHub/Azure DevOps tool for this. Grounding the review in the live PR metadata is required before reviewing code.

Parse the PR URL to extract:

- Organization (e.g., `episerveremea-expertservices`)
- Project (e.g., `First%20Mile`)
- Repository (e.g., `FirstMile`)
- PR ID (e.g., `8577`)

Fetch the PR details:

```bash
bun run scripts/remote-api.ts ado pr "<AZURE_DEVOPS_PR_URL>"
```

If this helper returns `401` or `403`, or returns `404` with a permission-style message, stop and ask the user to verify Azure DevOps access before continuing. Do not continue the PR review until the remote PR fetch succeeds.

**Verify** the target branch is `develop`. If not, warn the user.

#### Local pre-PR review mode

Determine the exact review scope before collecting the diff:

1. Confirm the base branch. Default to `develop` unless the user specifies another branch.
2. Determine whether you are reviewing:
    - the current branch against the base branch,
    - only staged changes,
    - only unstaged changes, or
    - the full worktree (staged + unstaged).
3. Capture the current branch name and worktree status:

```powershell
git branch --show-current
git status --short
```

If the user asks for a review of local uncommitted work, do not require a PR URL and do not block on Azure DevOps metadata.

### 2.2 Sync Local Branches

For both review modes, ensure the base branch used for comparison is current before trusting the diff.

1. Ensure local base branch is up to date:

   ```powershell
   git fetch origin <baseBranch>:<baseBranch>
   ```

2. For PR review mode, fetch the source branch locally:

   ```powershell
   git fetch origin <sourceBranch>:<sourceBranch>
   ```

### 2.3 Get the Diff

Use the narrowest diff command that matches the requested review scope.

#### PR review mode or committed local branch review

Use `git diff` with the three-dot notation to see changes introduced by the source branch:

```powershell
git diff <baseBranch>...<sourceBranch> --stat
git diff <baseBranch>...<sourceBranch>
```

Read the full context of changed files on the feature branch when needed:

```powershell
git show <sourceBranch>:<filePath>
```

#### Local staged-only review

```powershell
git diff --cached --stat
git diff --cached
```

#### Local unstaged-only review

```powershell
git diff --stat
git diff
```

#### Local full-worktree review

Collect both staged and unstaged diffs. Review them together and call out which findings come from staged vs unstaged changes if that distinction matters.

```powershell
git diff --cached --stat
git diff --cached
git diff --stat
git diff
```

When reviewing local changes, read the working tree version of changed files directly from disk rather than from `git show`.

### 2.4 Fetch Jira Tickets

Fetch ticket details and comments **only** via `scripts/remote-api.ts` (see ⛔ MANDATORY block at top of this file). Do not hit `atlassian.net` directly and do not use `Invoke-WebRequest`, `curl`, or any other tool for Jira data. Live ticket data is required before validating requirements or acceptance criteria.

Parse ticket keys from the provided URLs (e.g., `FMI-934` from `https://episerver-services.atlassian.net/browse/FMI-934`).

Fetch each ticket:

```bash
bun run scripts/remote-api.ts jira issue "<TICKET_KEY_OR_URL>" "summary,description,status,issuetype,comment"
```

**Important**: Fetch and read ALL comments on the ticket. Comments have **higher priority** than the ticket description when determining requirements.

To fetch comments separately if needed:

```bash
bun run scripts/remote-api.ts jira comments "<TICKET_KEY_OR_URL>"
```

If either Jira request returns `401` or `403`, or returns `404` with `Issue does not exist or you do not have permission to see it`, stop and ask the user to verify Jira access before continuing. Do not continue the review with stale or partial ticket data.

### 2.5 Adversarial Review

Adopt an **adversarial mindset**: your job is to break the code, not confirm it works. Assume every change contains at least one latent defect and hunt for it.

#### Phase 1 — Requirement Mismatch Attack

1. Extract **acceptance criteria**, **test data**, and **expected behavior** from the ticket description and comments.
2. For each criterion, attempt to construct a scenario where the code produces the wrong result or silently does nothing.
3. Look for requirements that are addressed in name only — where the code path exists but the behavior diverges from what the ticket actually specifies.

#### Phase 2 — Fault Injection (Mental Fuzzing)

For every changed method or code path, ask:

- **Null / empty inputs**: What happens if any parameter, collection, or config value is null, empty, or whitespace?
- **Boundary values**: What about zero, negative, max-int, empty arrays, single-element lists, strings at max length?
- **Concurrency**: Can two requests hit this path simultaneously and corrupt shared state?
- **Ordering**: Does this assume a specific call order that isn't enforced?
- **External failures**: What if an API call, DB query, or file read throws? Is there a silent swallow?

#### Phase 3 — Blast Radius Analysis

1. Trace every changed public interface upstream and downstream. Identify callers that were not updated.
2. Look for implicit contracts (e.g., a method that used to return non-null now can return null).
3. Check if removed or renamed symbols leave dead references in views, JSON contracts, or config.
4. Verify feature flags, DI registrations, and route registrations are consistent with the change.

#### Phase 4 — Security Adversarial Pass

- **Injection**: Can user-controlled input reach SQL, HTML, or command execution without sanitization?
- **AuthZ/AuthN**: Does the change accidentally expose data or actions to unauthorized users?
- **Information leakage**: Do error messages, logs, or API responses expose internals?
- **IDOR**: Can a user manipulate IDs to access another tenant's or user's data?

#### Phase 5 — Specification Completeness

- List every acceptance criterion and mark it ✅ proven-covered or ❌ not-covered/partially-covered.
- Flag any behavior that is implemented but has **no corresponding test** — treat untested logic as suspect.
- Identify any ticket requirement that is entirely missing from the diff.

### 2.6 Summarize Findings

Present the review as:

1. **Review Overview**: PR title/author/branch info when available, or local branch/worktree scope when reviewing pre-PR changes, plus files changed.
2. **Ticket Context**: Brief summary of what the ticket(s) require.
3. **Changes Summary**: What each file change does.
4. **Adversarial Findings**: For each finding, state:
    - The **attack vector** (how you tried to break it)
    - The **evidence** (file:line citation and reproduction scenario)
    - The **impact** (what goes wrong if this defect ships)
5. **Issues Table**: Categorized by severity:
    - **Critical/Bug**: Will cause runtime errors, data loss, or security breach.
    - **High**: Doesn't meet spec requirements, risks data corruption, or has no test coverage for critical logic.
    - **Medium**: Potential issues under specific conditions that should be clarified or defended.
    - **Low**: Code quality, minor improvements, defensive hardening.
    - **Info**: Observations, compiler warnings, style.
6. **Acceptance Criteria Checklist**: Every criterion marked ✅ or ❌ with file:line proof.
7. **Verdict**: Approve / Request changes / Needs clarification.

> **Verdict bias**: Default to "Request changes" unless you can prove correctness for all critical paths. The burden of proof is on the code, not the reviewer.

### 2.7 Offer Fixes and Unit Tests

For each identified issue:

1. Propose a concrete code fix.
2. Provide relevant unit tests following the project conventions (see the `unit-tests` skill for test conventions).

Unit test file location mirrors the source structure under `<project>.Tests/`:

```
Source:  FirstMile.Services/Helpers/OrderCreationHelper.cs
Test:    FirstMile.Services.Tests/Helpers/OrderCreationHelperTests.cs

Source:  firstmile.web/Api/SavedBasketController.cs
Test:    firstmile.web.Tests/Api/SavedBasketControllerTests.cs
```

Treat wrong test project placement as a review finding. If a test targets one assembly but lives under another assembly's test project, request changes. If the matching test project does not exist, require creating it rather than accepting the wrong destination.

Reference `FirstMile.Services.Tests/Email/EmailServiceTests.cs` for test style examples.

## Step 3: Apply Changes

After presenting the review and proposed fixes:

1. Ask the user to confirm which fixes to apply.
2. Apply the approved changes to the reviewed branch or local worktree files.
3. **Do NOT push automatically.** Remind the user to review the changes carefully before pushing to the remote.

## Notes

- If anything is unclear about the requirements or the code, **ask the user** — do not make assumptions. Refer to `.ai/standards/stakeholders.md` for escalation guidance (PO for requirements, Rob Paine for Salesforce, Tuyen Pham for architecture).
- When the diff is large, focus on the most impactful changes first.
- Always verify string literals, magic values, and status codes against the Jira ticket spec.
- Pay special attention to Salesforce API field values — typos in status strings or field names will cause silent failures.
