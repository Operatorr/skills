# Review Rules — Project Standards

**This file defines the code review standards for this repository.**

The GitHub PR Reviewer skill treats the rules in this file as **highest priority**. All reviews will enforce these standards strictly.

---

## Core Principles

- **Clarity and maintainability first** — Code should be easy for the whole team to understand and modify.
- **Fail fast and explicitly** — Prefer early validation, clear errors, and defensive programming.
- **Consistency over cleverness** — Follow existing patterns in the codebase unless there is a strong reason not to.
- **Test what matters** — New logic and changes to critical paths must include meaningful tests.

## Mandatory Rules (Block Merge)

These rules are non-negotiable. Violations should be fixed before merging.

### 1. Null / Undefined Safety
- Never access properties or call methods on a value that can be `null` or `undefined` without an explicit guard.
- Check the result of any function that can return `null`/`undefined` **immediately** after the call.
- Use optional chaining (`?.`) and nullish coalescing (`??`) where appropriate, but prefer explicit checks for critical logic.

### 2. Error Handling
- All errors must be handled explicitly. Never let unhandled promises or exceptions crash the process.
- Use custom error classes (from `src/errors/` or equivalent) instead of throwing generic `Error` or `new Error()`.
- Log errors with sufficient context, but **never log sensitive data** (passwords, tokens, PII, secrets).

### 3. Security
- All user input must be validated and sanitized before use in database queries, templates, or system commands.
- Use parameterized queries / ORM methods. Never build SQL or shell commands via string concatenation.
- Authentication and authorization checks must happen **before** any business logic executes.
- Secrets must only come from environment variables or a secrets manager — never hardcoded.

### 4. Input Validation & APIs
- All public API endpoints and functions must validate input using shared validation schemas.
- New public interfaces (functions, classes, API routes) **must** include clear documentation (JSDoc, TSDoc, or equivalent).
- Response formats must follow the project’s standard envelope when applicable.

### 5. Testing Requirements
- Every new feature, bug fix, or change to critical logic **must** include tests (happy path + at least one error/edge case).
- Use the project’s existing test utilities and patterns.
- Integration tests are required for changes involving external services or the database.

### 6. Performance & Scalability
- Avoid N+1 database queries. Use batch loading, joins, or caching where appropriate.
- New endpoints or background jobs that touch the database or external services should be reviewed for performance impact.

## Strongly Preferred Rules

These rules improve long-term maintainability. Flag them unless there is a justified exception.

- Prefer early returns over deeply nested conditionals.
- Use `const` by default. Only use `let` when reassignment is genuinely required.
- Replace magic numbers and strings with named constants.
- Extract duplicated logic into shared utilities or helper functions.
- Keep functions focused and reasonably sized (ideally under 50–60 lines for complex logic).
- Use descriptive names — avoid cryptic abbreviations except for well-known ones (`id`, `url`, `ctx`).

## Language & Framework Specific (Add as Needed)

### TypeScript / JavaScript
- Enable and respect strict null checks.
- Avoid `any` unless absolutely necessary and documented.
- Prefer composition over deep class inheritance.

### Python
- Follow the project’s Black + Ruff configuration.
- Use type hints on all public functions and important internal functions.
- Prefer `pydantic` models for data validation and settings.

### Go
- Follow `golangci-lint` rules (enforced in CI).
- Use `errors.Is` and `errors.As` for error handling.
- Keep functions small and focused.

---

## What NOT to Flag

Do **not** flag these unless the rule is explicitly added to this file:

- Minor formatting or style preferences already handled by linters
- Personal coding style differences (“I would have written this differently”)
- Suggestions for large refactors unless the PR itself is a refactoring effort
- Adding comments to self-explanatory code
- Performance micro-optimizations without clear evidence of a bottleneck

---

## How to Customize This File

1. Edit this file directly in your repository.
2. Be as **specific and concrete** as possible — vague rules lead to inconsistent reviews.
3. Add project-specific sections (e.g., “Payment Service Rules”, “Frontend Component Rules”).
4. After a few weeks of reviews, review the comments the AI made and update this file with new patterns you want enforced.

**Tip**: The more detailed and opinionated this file is, the better and more consistent the automated reviews will become.

---

**Last updated**: May 2026  
**Maintained by**: Your Team

---

*This file powers the github-pr-reviewer skill. Keep it in the root of your repository as `REVIEW.md`.*