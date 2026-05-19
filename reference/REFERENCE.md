# GitHub PR Reviewer — Reference Materials

This file contains detailed templates, examples, and guidelines to support the main `SKILL.md`.

## 1. Full Handoff Prompt Template (Copy-Paste Ready)

Use this exact structure for every issue:

```markdown
**Handoff Prompt (copy-paste ready for Claude/Cursor/etc.):**
```
In the file `RELATIVE_PATH`, around lines START-END:

[Detailed instruction paragraph 1: What to change and why, referencing the specific problem]

[Detailed instruction paragraph 2: Exact implementation steps, including any null checks, error handling, or edge cases]

[Detailed instruction paragraph 3: How the surrounding code should look after the change. Reference similar patterns elsewhere in the codebase if relevant, e.g. "Follow the same pattern used in UserService.validateEmail()"]

After making the change:
- Verify the fix by [specific test command or manual check]
- Ensure no other call sites are broken
- Run the project's linter on the file

Only modify the necessary lines. Do not reformat unrelated code.
```
```

**Example of an excellent handoff prompt** (for a null-check issue):

```
In the file `src/services/AttendeeService.ts`, around lines 55-78:

After calling `AttendeeService.getAttendeeByAttendeeId(eventId, attendeeIdCode)`, immediately check if the returned value is null or undefined. If it is, return a 404 response with JSON body `{ error: "Attendee not found" }` instead of continuing execution.

Only access `attendee.id`, `attendee.name`, `attendee.email`, or `attendee.company` **after** the null check passes.

If `getAttendeeActivitiesDetailed` expects the internal `attendee.id` (not `attendeeIdCode`), call it with `attendee.id` after the check.

Follow the exact error response pattern used in `EventController.getEventById` (lines 120-128 in the same file).

After the change, add a unit test case for the "attendee not found" scenario in `AttendeeService.test.ts`.
```

## 2. Severity & Category Guidelines

| Severity | When to Use | Example Issues |
|----------|-------------|----------------|
| **Critical** | Immediate data loss, security breach, production crash, auth bypass | SQL injection, missing auth on sensitive endpoint, unhandled promise rejection that crashes process |
| **High** | Logic bug that breaks core feature, missing critical error handling, major performance regression | Wrong calculation in pricing engine, race condition in payment flow, N+1 query in hot path |
| **Medium** | Code smell that will cause maintenance pain, missing tests for new logic, inconsistent patterns | Duplicate validation logic, magic numbers, missing input sanitization on user-facing fields |
| **Low** | Minor style violation that team has explicitly standardized | Wrong import order, missing JSDoc on internal function (if rule exists in REVIEW.md) |

**Rule of thumb**: If you wouldn't block a merge for it in a real senior review, don't flag it as Medium or higher.

## 3. Sample Full Review Output (Abbreviated)

```markdown
## High Issue: Missing null check on attendee lookup can cause runtime error

**File:** `src/controllers/EventController.ts:62-75`
**Category:** Bug

**Problem:**
`getAttendeeByAttendeeId` can return null when the attendee does not exist. The code immediately accesses `attendee.id` without checking, leading to a potential 500 error instead of a proper 404.

**Impact:**
Users will see generic server errors instead of clear "not found" messages. This also masks real data issues.

**Suggested Fix:**
```diff
 const attendee = await AttendeeService.getAttendeeByAttendeeId(eventId, attendeeIdCode);
+if (!attendee) {
+  return { status: 404, body: JSON.stringify({ error: "Attendee not found" }) };
+}
 const activities = await AttendeeService.getAttendeeActivitiesDetailed(eventId, attendeeIdCode);
```

**Handoff Prompt (copy-paste ready for Claude/Cursor/etc.):**
```
In the file `src/controllers/EventController.ts`, around lines 62-75:

After the line `const attendee = await AttendeeService.getAttendeeByAttendeeId(eventId, attendeeIdCode);`, add an immediate null/undefined check. If attendee is falsy, return a 404 response with JSON body `{ error: "Attendee not found" }`.

Only proceed to call `getAttendeeActivitiesDetailed` and access attendee properties after the check passes.

Use the exact same error response format as the one in `getEventById` (around line 125).

Add a corresponding test case for the "attendee not found" path.
```

---

## Medium Issue: Inconsistent error handling pattern

**File:** `src/utils/validation.ts:34`
**Category:** Maintainability

... (additional issues)

### PR Summary
This PR adds attendee activity tracking to the event endpoint. The core logic is sound and follows existing patterns well. Two important issues were found (one High, one Medium) that should be fixed before merge. Overall a clean contribution.

### Recommendations
- [x] Fix the null-check issue (High)
- [ ] Add test coverage for the new attendee lookup path
- [ ] Consider extracting the attendee resolution into a shared utility

### Statistics
- Files changed: 4
- Issues found: 3 (Critical: 0, High: 1, Medium: 1, Low: 1)
```

## 4. GitHub CLI Commands for Posting Reviews

**Best practice for rich reviews (recommended):**

```bash
# Create a review with body from file (supports markdown)
gh pr review 123 --comment --body-file /tmp/review.md

# For line-specific comments (more advanced)
gh api repos/OWNER/REPO/pulls/123/reviews \
  -f event=COMMENT \
  -f body="Overall review summary here" \
  -f comments='[{"path":"src/file.ts","line":42,"body":"Specific comment here"}]'
```

**Simple comment fallback:**
```bash
gh pr comment 123 --body-file /tmp/review.md
```

## 5. Common Pitfalls to Avoid

- **Do not** flag every minor formatting issue unless `REVIEW.md` explicitly requires it.
- **Do not** suggest massive refactors in a review unless the PR itself is a refactor.
- **Always** provide a working diff or code snippet — never just describe the problem.
- **Never** include time-sensitive information (dates, specific PR numbers in templates).
- When the PR is excellent: Still write 2-3 sentences of positive feedback. This builds trust.

## 6. Extending This Skill

To make the reviewer even stronger:

1. Add a `scripts/` folder with helper scripts (e.g., `parse-diff.js` or `format-review.js`).
2. Create sub-skills:
   - `security-reviewer` (focused mode)
   - `performance-reviewer`
   - `test-coverage-analyzer`
3. Integrate with your issue tracker (Linear/Jira) by reading ticket context in Phase 1.

---

**This reference material makes the reviewer skill repeatable and high-quality across different codebases and team standards.**
