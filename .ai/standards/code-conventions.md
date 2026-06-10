# Code Conventions — V1.0

<!-- Fill: extend or replace these rules after repo-scan to match your stack and language -->

## TypeScript (if applicable)

- Strict mode always on. `any` only when unavoidable — must have inline comment explaining why.
- Prefer `unknown` over `any` for external or untyped data.
- Export types from a central shared types module — no scattered inline exports across modules.
- No `Promise.all` for operations with sequential dependencies — use sequential `await`.

<!-- Add or replace with language-specific type rules if not TypeScript -->

## Naming

| Artifact | Convention |
|---|---|
| Files | kebab-case (`user.service.ts`, `auth.handler.go`) |
| Classes | PascalCase |
| Functions / variables | camelCase |
| Constants | UPPER_SNAKE_CASE |
| DB columns | snake_case |
| URL paths | kebab-case |

<!-- Fill: adjust naming conventions per language and framework after repo-scan -->

## Module structure

- Each module has a single entry point (`<module>.service.ts`, `index.ts`, or equivalent).
- No circular imports between modules.
- Domain logic in service layer — not in controllers, handlers, or background jobs.
- Background workers and job handlers dispatch to services — no business logic in job files.
- Shared module exports types and utils only — no business logic in shared.

## Error handling

| Context | Rule |
|---|---|
| API handlers | Return structured error `{ error, message, statusCode }`. Log full stack server-side. Never expose stack trace in response. |
| Background jobs | Log `{ jobId, error, stack }` on every failure. Never swallow errors. |
| General | No naked `throw` at module boundary. No empty catch blocks. |

<!-- Fill: add project-specific error handling patterns after repo-scan -->

## Logging

- Include a correlation ID (e.g. request ID, job ID, entity ID) on every log in request/job context.
- Structured logs (JSON) — no `console.log` or unstructured print in production code paths.
- Log levels: `info` for state transitions, `error` for failures, `debug` for verbose output.

## Comments

Write a comment only when WHY is non-obvious: hidden constraint, subtle invariant, specific bug workaround.
Never explain what the code does. Never reference the task, PR, or callers.

## Abstraction limits

- Three similar lines is better than a premature abstraction.
- No helper extraction unless used in 3+ places.
- No feature flags or backwards-compat shims — change the code directly.

## Stack-specific rules

<!-- Fill after repo-scan — add rules specific to this project's framework, ORM, queue system, etc. -->
<!-- Examples: -->
<!-- - No raw SQL string interpolation — parameterized queries or ORM always -->
<!-- - No direct DB writes from worker/job files -->
<!-- - No <framework-specific anti-pattern> -->