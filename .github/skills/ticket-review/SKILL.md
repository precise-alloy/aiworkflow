---
name: ticket-review
description: 'Analyze Jira tickets and produce an execution plan with detailed task lists. USE FOR: breaking down Jira tickets into implementation tasks, identifying files to modify, planning unit tests, and updating documentation. DO NOT USE FOR: actually implementing the changes, creating PRs, or deploying code.'
argument-hint: 'Provide the Jira ticket URL(s) and any additional context'
---

# Ticket Review & Execution Planning

Analyze Jira tickets, understand requirements, inspect the codebase, and produce a detailed execution plan saved to `docs/src/tickets/<ticket-id>-review.md`.

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

1. Read `.ai/AGENTS.md` — follow golden rules and pre-coding read order.
2. **Always start from Step 1** (Gather Input) — ask the user for ticket URL(s) and context.
3. **After gathering input, always run Step 0** (Check Existing Work) — check for existing review/task files before proceeding to full analysis.
4. After producing or updating the review document, triage per `.ai/workflows/generate-tasks.md` and generate task files in `.ai/tasks/{branch-slug}/`.
5. When generating task files, check `.ai/tasks/` for existing files matching this ticket — do NOT create duplicates (update them instead).

### Triage output:

- **TRIVIAL/SIMPLE**: Review document only — no task file needed.
- **STANDARD**: Review document + full task file in `.ai/tasks/`.
- **EPIC**: Review document + full task file + `.ai/memory/{branch-slug}-{feature-slug}-context.md`.

### Documentation planning (mandatory):

When generating task files, always include a DOC UPDATE section that specifies:

1. Which documentation files need to be created or updated.
2. What content should be added/changed (with ticket ID for traceability).
3. Ensure task STEPS include explicit instructions to add detailed comments with the ticket ID in all code changes.

This ensures documentation stays current and future requirements can build on accurate information.

### QA mode (`task`):

- For STANDARD/EPIC: generate `{task-name}.qa.md` alongside each task file.
- QA doc should contain: affected features, test scenarios, regression areas from §4 of the review.

### Standards check:

- Load `.ai/standards/testing-policy.md` — ensure unit test plan in §3.7 matches required test types.
- Load `.ai/standards/security.md` — if ticket touches auth/billing/tenant isolation.
- Load `.ai/standards/definition-of-done.md` — DONE WHEN in task files must satisfy these gates.
- Load `.ai/standards/stakeholders.md` — know who to escalate questions to and who can confirm decisions.

## Prerequisite

Ensure the user has a `.env.local` file at the workspace root with the following keys populated:

```dotenv
JIRA_PAT=
JIRA_EMAIL=
AZURE_DEVOPS_READONLY_PAT=
AZURE_DEVOPS_EMAILS=
```

These PATs are the required source of truth for remote Jira and Azure DevOps data. The remote-api helper fetches remote ticket details from Jira and any referenced remote PR metadata from Azure DevOps using PAT-backed REST API calls instead of relying on cached summaries or local assumptions.

**Before proceeding, always run the sanctioned env check via Bun. Do not use `Test-Path`, `Get-Content`, `cat .env.local`, or any other ad-hoc check — results differ across shells and lead to false positives.**

```bash
bun run scripts/remote-api.ts check-env
```

Interpret the result:

- **Exit 0** (`ok: true`) — file exists and all four keys are present and non-empty. Proceed.
- **Exit 1 with `envLocalExists: false`** — create `.env.local` at the workspace root with the four keys shown above (empty values are fine), then tell the user to fill them in and re-run the skill.
- **Exit 1 with `missing: [...]`** — tell the user which keys are missing/empty and ask them to fill those values, then re-run the skill.

When using either PAT during the skill, treat any authentication or authorization failure (`401`, `403`, invalid credentials, PAT expired/revoked) as a hard stop for remote analysis. Ask the user to provide a valid PAT for the failing system, then retry the remote fetch before continuing.

## Step 0: Check Existing Work

After gathering ticket URL(s) in Step 1, **always** check for existing review and task files before proceeding:

1. Check if `docs/src/tickets/<TICKET_ID>-review.md` exists.
2. Check if `.ai/tasks/` contains any task files matching this ticket ID.
3. Check if `.ai/memory/` contains any context files matching this ticket ID.

### If existing files are found → Incremental Update

When a review file already exists, perform an incremental update instead of a full re-review:

1. Read the existing review file and note the `Last Reviewed` datetime (UTC).
2. Fetch the ticket's change history (changelog) since that datetime using the helper (see ⛔ MANDATORY block at top of this file):
   ```bash
   bun run scripts/remote-api.ts jira changelog "<TICKET_KEY_OR_URL>"
   ```
3. Fetch comments via the helper and check for any added or updated after the last review date:
   ```bash
   bun run scripts/remote-api.ts jira comments "<TICKET_KEY_OR_URL>"
   ```
4. If there are **no changes** since last review — inform the user that the review is up-to-date and ask if they want a full re-review anyway.
5. If there **are changes**, identify:
    - **Description updates** — requirements may have changed.
    - **New comments** — may contain answers to open questions, confirmation of assumptions, or additional context.
    - **Status changes** — ticket may have moved to a different state.
    - **Acceptance criteria updates** — scope may have shifted.
6. Reflect all relevant changes into the review document and associated task files:
    - Move answered questions from "Open Questions" to "Decisions".
    - Update assumptions that have been confirmed or invalidated.
    - Adjust task lists if scope changed.
    - Update the `Last Reviewed` datetime to the current UTC time.
7. Present a **diff summary** to the user showing what changed in the review.

If the changelog or comment fetch fails because the Jira PAT is invalid, expired, revoked, or lacks access, stop and ask the user to provide a valid Jira PAT before continuing the incremental update.

### If no existing files are found → Full Review

Proceed to Step 2 (Analyze Requirements & Codebase) and perform the full review workflow.

## Step 1: Gather Input

Ask the user for:

1. The Jira ticket URL(s) (e.g., `https://episerver-services.atlassian.net/browse/FMI-934`)
2. Any additional information or context that is not captured in the ticket (e.g., verbal decisions, Slack discussions, architectural constraints, priority notes)

## Step 2: Analyze Requirements & Codebase

### 2.1 Fetch Jira Ticket(s)

Fetch Jira ticket details and comments **only** via `scripts/remote-api.ts` (see ⛔ MANDATORY block at top of this file). Do not hit `atlassian.net` directly and do not use `Invoke-WebRequest`, `curl`, or any other tool for Jira data. Live ticket data is required before producing the review document or execution plan.

Parse ticket keys from the provided URLs (e.g., `FMI-934` from `https://episerver-services.atlassian.net/browse/FMI-934`).

Fetch each ticket:

```bash
bun run scripts/remote-api.ts jira issue "<TICKET_KEY_OR_URL>" "summary,description,status,issuetype,comment,acceptance_criteria"
```

Fetch all comments separately:

```bash
bun run scripts/remote-api.ts jira comments "<TICKET_KEY_OR_URL>"
```

If either Jira request returns `401` or `403`, or returns `404` with `Issue does not exist or you do not have permission to see it`, stop and ask the user to verify Jira access before continuing. Do not continue planning with stale or partial ticket data.

If the ticket references an Azure DevOps PR or branch that affects implementation planning, fetch that PR via `bun run scripts/remote-api.ts ado pr "<PR_URL>"` (see ⛔ MANDATORY block at top of this file) before finalizing the plan. If the Azure DevOps call fails because the PAT is invalid, expired, revoked, or lacks access, ask the user to provide a valid Azure DevOps PAT before continuing.

**Important**: Comments have **higher priority** than the ticket description when determining requirements. If a comment contradicts or refines the description, the comment takes precedence.

### 2.2 Identify Requirements

From the ticket description and comments, extract:

1. **Acceptance criteria** — what must be true for the ticket to be considered done.
2. **Expected behavior** — how the feature/fix should work.
3. **Test data** — any specific values, scenarios, or edge cases mentioned.
4. **Constraints** — performance, security, compatibility notes.

### 2.3 Analyze Related Code

Based on the requirements, search the codebase to identify:

- **Existing code** that implements related functionality.
- **Entry points** (controllers, API endpoints, blocks, pages) affected.
- **Services and helpers** that will need changes.
- **Models and DTOs** that may need updates.
- **Configuration** files or constants involved.

Use semantic search, grep, and file exploration to locate relevant files across:

- `FirstMile.Services/` — business logic, helpers, services
- `firstmile.web/` — controllers, API endpoints, view components
- `FirstMile.Models/` — data models, blocks, pages, enums
- `FirstMile.Integration/` — external integrations
- `FirstMile.Salesforce/` — Salesforce-related logic
- `FirstMile.WebUtils/` — shared web utilities

### 2.4 Check for UI Changes

If the ticket involves UI updates (new components, layout changes, styling, frontend behavior):

1. Inspect `firstmile.ui/` for relevant frontend source files (scripts, styles, components).
2. Inspect `firstmile.widgets/` for widget definitions and configurations.

**Note**: Ignore `firstmile.patterns/` — this folder contains auto-generated build artifacts produced by the UI build command and should not be edited directly.

### 2.5 Review Existing Documentation

Check the `docs/` folder for any existing documentation related to the feature area:

```
docs/
  branching.md
  db-and-logs.md
  git-hooks.md
  README.md
  src/
```

Note which docs are relevant and whether they need updating or if new documentation should be created.

### 2.6 Ask Clarifying Questions

If anything is unclear, ambiguous, or needs confirmation:

- **Ask the user clearly and specifically.** Do not make assumptions.
- Refer to `.ai/standards/stakeholders.md` to identify who should answer each question:
    - Requirements/acceptance criteria → **Caiti Black** (PO)
    - Salesforce fields/statuses/API → **Rob Paine** (SF Admin)
    - Architecture/implementation → **Tuyen Pham** (Solution Architect)
- The user may answer directly, or may need to escalate to the appropriate stakeholder.
- Frame questions so they can be forwarded to non-technical stakeholders if needed.
- Clearly distinguish between questions that block implementation vs. questions that are nice-to-have clarifications.

### 2.7 Raise Concerns

Proactively raise any concerns related to:

- **Security** — authentication, authorization, input validation, data exposure, injection risks.
- **Compliance** — GDPR, data retention, PII handling, audit logging.
- **Performance** — N+1 queries, large payloads, missing caching, scalability.
- **Breaking changes** — API contract changes, backward compatibility.
- **Data integrity** — race conditions, partial updates, migration risks.

These must be clearly documented in the output so they can be reviewed before implementation begins.

## Step 3: Create Review Document

Create the file `docs/src/tickets/<TICKET_ID>-review.md` (e.g., `docs/src/tickets/FMI-934-review.md`) with the following structure:

````markdown
# <TICKET_ID>: <Ticket Summary>

> Last Reviewed: <YYYY-MM-DD HH:mm UTC>  
> Status: <ticket status>  
> Type: <issue type>

## 1. Questions, Assumptions & Decisions

### Open Questions (Needs Answer)

Items below may need product owner confirmation before the dev team can provide estimation.

- [ ] <Clear question that needs PO/stakeholder answer>
- [ ] <Another question>

### Assumptions

- <Assumption made and reasoning>

### Decisions

- <Decision made based on ticket comments or user input>

## 2. Proposed Implementation

### Approach

<Brief description of the overall technical approach and rationale.>

### Solution Details

<Detailed explanation of how the implementation will work, including:>

- Architecture decisions
- Data flow (use Mermaid diagrams when visual representation aids understanding)
- Integration points
- Error handling strategy

### Diagrams (if applicable)

When the implementation involves complex flows, component interactions, or state transitions, include Mermaid diagrams to visualize them. Use ```mermaid fenced code blocks.

Examples of when to include diagrams:

- Sequence diagrams for multi-step API/service interactions
- Flowcharts for complex decision logic
- State diagrams for status/workflow transitions
- Class diagrams for new model relationships

## 3. Detailed Task List

### 3.1 Models & Configuration

| #   | File Path         | Action          | Description                  |
| --- | ----------------- | --------------- | ---------------------------- |
| 1   | `path/to/file.cs` | Modify / Create | What needs to change and why |

### 3.2 Services & Business Logic

| #   | File Path | Action | Description |
| --- | --------- | ------ | ----------- |

### 3.3 Integration

| #   | File Path | Action | Description |
| --- | --------- | ------ | ----------- |

### 3.4 Controllers & Endpoints

| #   | File Path | Action | Description |
| --- | --------- | ------ | ----------- |

### 3.5 UI & Frontend

| #   | File Path | Action | Description |
| --- | --------- | ------ | ----------- |

### 3.6 Wiring & DI

| #   | File Path | Action | Description |
| --- | --------- | ------ | ----------- |

### 3.7 Unit Tests

| #   | Test File Path                   | Tests to Add                         | Covers            |
| --- | -------------------------------- | ------------------------------------ | ----------------- |
| 1   | `Project.Tests/path/TestFile.cs` | `MethodName_Scenario_ExpectedResult` | Brief description |

Test file location convention:

```
Source:  FirstMile.Services/Helpers/OrderCreationHelper.cs
Test:    FirstMile.Services.Tests/Helpers/OrderCreationHelperTests.cs

Source:  firstmile.web/Api/SavedBasketController.cs
Test:    firstmile.web.Tests/Api/SavedBasketControllerTests.cs
```

### 3.8 Documentation

| #   | Doc File Path          | Action          | Description      |
| --- | ---------------------- | --------------- | ---------------- |
| 1   | `docs/feature-name.md` | Create / Update | What to document |

Guidelines:

- Existing docs are intentionally brief — **expand them** with implementation details, usage examples, and architecture notes when touching related features.
- Create new docs for entirely new features or workflows.
- Include: purpose, how it works, configuration, edge cases, and troubleshooting tips.

## 4. QA Verification Notes

### Test Scenarios

| #   | Scenario        | Steps                       | Expected Result      |
| --- | --------------- | --------------------------- | -------------------- |
| 1   | <Scenario name> | <Steps to reproduce/verify> | <What QA should see> |

### Edge Cases to Verify

- <Edge case 1>
- <Edge case 2>

### Regression Areas

- <Area that might be affected and should be regression-tested>

### Test Data Requirements

- <Any specific data setup needed for QA>

## 5. Risks & Concerns

### Security

- <Security concern if any, or "None identified">

### Compliance

- <Compliance concern if any, or "None identified">

### Performance

- <Performance concern if any, or "None identified">

### Breaking Changes

- <Breaking change risk if any, or "None identified">
````

## Step 4: Generate Task Files (AI Workflow)

After the review document is saved, generate task files per the `.ai/workflows/generate-tasks.md` format:

1. **Triage** the ticket using blast radius, contract impact, and validation clarity.
2. **Create branch slug** from the planned branch name: `feat/FMI-{id}-{slug}` → `feat-FMI-{id}-{slug}`.
3. **Write task file(s)** to `.ai/tasks/{branch-slug}/{NNN-name}.md` using the STANDARD/EPIC template.
4. **Write QA file(s)** to `.ai/tasks/{branch-slug}/{NNN-name}.qa.md` (QA mode is `task`).
5. **For EPIC**: create `.ai/memory/{branch-slug}-{feature-slug}-context.md` with decisions and progress.

### Task file DONE WHEN must include:

- All acceptance criteria from ticket
- `[ ] Compiles without errors`
- `[ ] Unit tests pass (per testing-policy.md)`
- `[ ] No files outside CONTEXT modified`
- `[ ] No claim made about existing code without citing file:line`

### Present execution plan:

After generating task files, output a summary:

- Triage level
- Task file paths created
- Suggested execution order
- Which executor to use (Claude Code / GitHub Copilot / Codex)

## Notes

- The `Last Reviewed` datetime (UTC, format `YYYY-MM-DD HH:mm UTC`) is **mandatory** in every review file. It enables efficient re-reviews by scoping changelog/comment checks to only what changed since last review — even multiple re-reviews on the same day.
- On re-review, always check ticket change history and comments for: answers to previously open questions, confirmation/rejection of assumptions, scope changes, and new context. Update the review and task files accordingly.
- If anything is unclear about the requirements, **ask the user** — do not make assumptions. The user can escalate to the product owner for confirmation.
- When the ticket is large, suggest splitting into smaller sub-tasks if appropriate.
- Pay attention to Salesforce field names and status strings — typos cause silent failures.
- Consider backward compatibility when modifying shared models or APIs.
- The review document should be actionable enough that another developer (or the `unit-tests` skill) can follow it without re-reading the ticket.
- Security and compliance concerns must always be raised explicitly — never silently ignore them.
- Reference `FirstMile.Services.Tests/Email/EmailServiceTests.cs` for test style examples.
