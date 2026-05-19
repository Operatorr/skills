---
name: github-pr-reviewer
description: Performs deep, context-aware code reviews on GitHub Pull Requests. Analyzes changes against the full codebase, identifies bugs, security issues, logic errors, and style violations, then provides structured feedback with exact fix suggestions and ready-to-use handoff prompts for other AI coding agents. Use when the user asks to review a PR, analyze pull request changes, run automated code review, or mentions PR #X or reviewing code changes.
---

# GitHub PR Reviewer

**Expert-level AI code reviewer** that matches or exceeds CodeRabbit quality by combining full repository context, multi-step analysis, static checks, and precise handoff prompts.

## Quick Start

```bash
# Interactive use in Claude Code
Review PR #123 with full context and post the review to GitHub

# Or for local changes (no PR yet)
Review the current branch changes against main as if it were a PR
```

The agent will:
1. Gather PR diff + full repo context
2. Perform deep analysis
3. Output a professional review with per-issue **handoff prompts**
4. (Optional) Post directly to GitHub using `gh`

## Core Workflow

Follow this process **exactly** for every review:

### Phase 1: Context Gathering (Mandatory)
- [ ] Use `gh pr view <number> --json` or `gh pr diff` to get title, description, files changed, and diff.
- [ ] If full repo context is needed (recommended for accuracy): `git clone` or checkout the PR branch + base branch.
- [ ] Read key project files for standards:
  - `REVIEW.md` (highest priority review rules)
  - `CLAUDE.md` or `.claude.md`
  - `CONTRIBUTING.md`, `coding-standards.md`, or similar
  - `package.json` / `pyproject.toml` / equivalent for tech stack
- [ ] Note linked issues/tickets from PR description or comments.

### Phase 2: Multi-Dimensional Analysis
Analyze **in this order** (do not skip):

1. **PR Intent** — What is the goal of this change? (Summarize in 2-3 sentences)
2. **Static & Tool Analysis** (if environment allows):
   - Run relevant linters (e.g., `eslint`, `ruff`, `golangci-lint`, `mypy`, etc.)
   - Security scanners if available (e.g., `semgrep`, `trivy`)
3. **Semantic & Logic Review** (agentic exploration):
   - Read changed files + surrounding context (use `read_file` or `grep` for patterns).
   - Check for: null/undefined handling, error paths, race conditions, performance regressions, missing tests, breaking changes, security issues (injection, auth, secrets), accessibility, i18n.
   - Cross-file impact: Does this change affect other modules? Are there similar patterns elsewhere that should be followed?
4. **Style & Consistency**:
   - Enforce rules from `REVIEW.md` / team standards.
   - Flag "slop" (unnecessary complexity, duplicated logic, poor naming).

**Severity Scale** (use consistently):
- **Critical**: Security vulnerability, data loss, runtime crash, major regression
- **High**: Logic bug, missing error handling, performance issue, broken feature
- **Medium**: Code smell, missing test, minor inconsistency, maintainability issue
- **Low/Nit**: Style preference, minor formatting (only flag if it violates explicit team rule)

### Phase 3: Structured Output
For **every issue** found, produce this exact format:

```markdown
## [Severity] Issue: [Short descriptive title]

**File:** `path/to/file.ext:line-range`
**Category:** Bug | Security | Logic | Maintainability | Testing | Style

**Problem:**
[Clear 1-2 sentence explanation of what's wrong and why it matters]

**Impact:**
[What could go wrong if left unfixed — be specific]

**Suggested Fix:**
```diff
[Exact unified diff or before/after code block]
```

**Handoff Prompt (copy-paste ready for Claude/Cursor/etc.):**
```
In the file `path/to/file.ext`, around lines X-Y:

[Very detailed, self-contained instruction that includes:
- Exact change needed
- Why the change is required (context from surrounding code)
- Any edge cases to consider
- How to verify the fix
- Reference to relevant project patterns if applicable]

Only modify the necessary lines. Keep the rest of the function/file unchanged unless the fix requires it.
```
```

**After all issues**, add:

### PR Summary
[2-4 sentence overview of the PR + overall assessment]

### Recommendations
- [ ] High-priority fixes to make before merge
- [ ] Suggested tests to add
- [ ] Optional improvements

### Statistics
- Files changed: X
- Issues found: Y (Critical: A, High: B, Medium: C, Low: D)

## Posting the Review to GitHub

**Preferred method (structured review with line comments):**
Use the GitHub CLI to create a proper review:

```bash
gh pr review <PR_NUMBER> --comment --body-file review.md
# or for line-specific comments, use the API or multiple --comment flags
```

**Alternative (simple comment):**
Post the full structured output as a single comment using:
```bash
gh pr comment <PR_NUMBER> --body-file review.md
```

Always ask the user before posting if in interactive mode.

## Advanced Features & Customization

### Team-Specific Rules (Highest Impact)
Create a `REVIEW.md` file in the repo root with rules like:
```markdown
# Review Rules for This Project

- All new public APIs must have JSDoc/TSDoc
- Prefer early returns over nested ifs
- Never access array[0] without length check
- Use our custom error classes (see src/errors/)
- All database queries must go through the Repository pattern
```

The reviewer skill will treat `REVIEW.md` as **highest-priority instructions**.

### Handling Large PRs (>20 files or >500 lines)
- Focus first on Critical/High issues only.
- Summarize low-severity items in a "Additional Notes" section.
- Suggest splitting the PR if it touches too many unrelated areas.

### Security-First Mode
When user says "security review" or "high security PR":
- Prioritize: auth bypass, injection, secrets, unsafe deserialization, SSRF, etc.
- Use stricter rules even for Medium issues.

### Integration with Other Skills
This skill works excellently with:
- `github-pr-creator` or feature planning skills
- Test generation skills
- Refactoring skills (feed handoff prompts directly to them)

## Best Practices for High-Quality Reviews

- Be **specific and actionable** — never say "this could be better".
- Always provide a **working code example** in the suggested fix.
- The handoff prompt must be **detailed enough** that another Claude instance can implement it without re-reading the whole file.
- Balance thoroughness with kindness — flag real problems, not personal style preferences.
- If no issues found: Still write a positive summary highlighting good practices observed.
- Learn from user feedback: After posting, note which comments were resolved vs dismissed to improve future reviews.

## Limitations & When to Escalate

- This skill excels at **code-level** issues. For architecture, product decisions, or complex distributed systems design, combine with human review.
- Very large monorepos may require manual file selection for performance.
- Always verify security findings with actual testing where possible.

**This skill turns Claude Code into a senior staff-level reviewer that never gets tired and always provides copy-paste-ready fixes.**
