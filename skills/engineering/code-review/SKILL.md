---
name: code-review
description: Perform deep, context-aware code reviews for GitHub pull requests or local branch changes. Use when the user asks to review a PR, analyze pull request changes, review current branch changes, run automated code review, or mentions PR #X.
---

# Code Review

Review code like a senior engineer: understand the intent, inspect the diff in context, run available checks, identify real risks, and produce actionable feedback with copy-paste-ready handoff prompts.

## Use Cases

```bash
Review PR #123 with full context
Review the current branch against main as if it were a PR
Do a security-focused review of this PR
Review these local changes but do not post to GitHub
```

## Principles

- Prioritize bugs, security issues, data loss, regressions, broken edge cases, missing tests, and maintainability risks.
- Do not nitpick personal style. Only flag style issues when they violate explicit project rules.
- Be specific and actionable. Every finding should include the exact file/location, impact, and a concrete fix.
- Review only by default. Do not edit code unless the user explicitly asks you to fix the issues.
- Never post to GitHub or another external system without asking the user first.

## Workflow

Follow this process for every review.

### Phase 1: Establish the Review Target

Determine whether the user wants to review:

1. A GitHub PR, e.g. `PR #123`
2. The current branch against a base branch, usually `main` or `master`
3. Uncommitted local changes
4. A specific set of files

For GitHub PRs, gather PR metadata and diff with `gh` when available:

```bash
gh pr view <number> --json title,body,author,baseRefName,headRefName,files,commits,url
gh pr diff <number>
```

For local branch reviews, inspect the merge base and diff:

```bash
git status --short
git branch --show-current
git merge-base HEAD origin/main 2>/dev/null || git merge-base HEAD main
git diff <merge-base>...HEAD
```

Adjust the base branch if the repository uses a different default branch.

### Phase 2: Gather Project Context

Read the changed files and enough surrounding code to understand the impact. Also check project standards in this priority order:

1. `REVIEW.md` in the target repository root, if present
2. `CLAUDE.md`, `.claude.md`, or equivalent agent instructions
3. `CONTRIBUTING.md`, coding standards, architecture docs, ADRs
4. Package/tooling files such as `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, etc.
5. Existing nearby tests and similar implementations

Treat the target repo's `REVIEW.md` as the highest-priority review rules.

If this skill's bundled reference files are available, use them as support material:

- `REFERENCE.md` — examples, output templates, severity guidance
- `PROJECT_REVIEW_TEMPLATE.md` — starter `REVIEW.md` users can copy into a project

### Phase 3: Run Available Checks

When safe and available, run relevant checks for the repository:

- Type checks
- Unit tests for changed areas
- Linters/format checks
- Security scanners if already configured

Do not install new tools or run destructive commands without permission. If checks are too expensive or unavailable, say so and continue with manual review.

### Phase 4: Analyze in Order

Analyze the change in this order:

1. **Intent** — What is the PR/change trying to accomplish?
2. **Correctness** — logic errors, invalid assumptions, broken edge cases, race conditions, bad state transitions
3. **Security** — auth/authz, injection, unsafe input handling, secrets, SSRF, XSS, unsafe deserialization, sensitive logging
4. **Reliability** — error handling, retries, idempotency, null/undefined handling, resource cleanup
5. **Data & API compatibility** — migrations, schema changes, response shapes, backwards compatibility
6. **Testing** — missing coverage for new logic, regression tests, edge cases, integration boundaries
7. **Performance** — N+1 queries, hot path regressions, unnecessary expensive work
8. **Maintainability** — unnecessary complexity, duplicated logic, poor naming, inconsistency with project patterns

For large PRs, focus first on Critical and High issues. Summarize lower-severity patterns instead of producing noisy line-by-line comments.

## Severity Scale

- **Critical** — security vulnerability, data loss, production crash, auth bypass, major regression
- **High** — logic bug, missing critical error handling, broken feature, serious performance issue
- **Medium** — missing test, maintainability risk, code smell likely to cause future bugs, minor behavior issue
- **Low/Nit** — explicit project style violation or small clarity issue

Rule of thumb: if you would not block a real merge for it, do not mark it Medium or higher.

## Output Format

Start with a short overall assessment. Then list findings from highest to lowest severity.

For every issue, use this format:

````markdown
## [Severity] Issue: [Short descriptive title]

**File:** `path/to/file.ext:line-range`
**Category:** Bug | Security | Logic | Reliability | Maintainability | Testing | Performance | Style

**Problem:**
[Clear 1-2 sentence explanation of what is wrong and why it matters.]

**Impact:**
[Specific consequence if this is not fixed.]

**Suggested Fix:**
```diff
[Exact unified diff, or a precise before/after code block if a diff is not practical.]
```

**Handoff Prompt:**
```
In the file `path/to/file.ext`, around lines X-Y:

[Detailed, self-contained instruction explaining the exact change needed, why it is required, edge cases to consider, and how to verify it. Reference relevant project patterns if applicable.]

Only modify the necessary lines. Keep unrelated code unchanged.
```
````

After all issues, include:

```markdown
### PR Summary
[2-4 sentence overview of the change and overall assessment.]

### Recommendations
- [ ] Highest-priority fixes before merge
- [ ] Tests to add or run
- [ ] Optional improvements

### Statistics
- Files changed: X
- Issues found: Y (Critical: A, High: B, Medium: C, Low: D)
- Checks run: [commands or "not run"]
```

If no issues are found, say so clearly and include a brief positive summary of what looked good.

## Posting to GitHub

If the user asks to post the review, ask for confirmation before posting.

Preferred simple review:

```bash
gh pr review <PR_NUMBER> --comment --body-file review.md
```

Fallback single comment:

```bash
gh pr comment <PR_NUMBER> --body-file review.md
```

For line-specific comments, use the GitHub API only when the user explicitly wants inline comments.
