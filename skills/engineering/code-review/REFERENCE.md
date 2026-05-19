# Code Review Reference

Supporting templates, examples, and guidelines for the `code-review` skill.

## Full Handoff Prompt Template

Use this structure for every issue:

````markdown
**Handoff Prompt:**
```
In the file `RELATIVE_PATH`, around lines START-END:

[What to change and why, referencing the specific problem.]

[Exact implementation steps, including null checks, error handling, or edge cases.]

[How the surrounding code should look after the change. Reference similar patterns elsewhere in the codebase if relevant.]

After making the change:
- Verify the fix by [specific test command or manual check]
- Ensure no other call sites are broken
- Run the project's linter/checks for the file

Only modify the necessary lines. Do not reformat unrelated code.
```
````

## Good Handoff Prompt Example

```markdown
In the file `src/services/AttendeeService.ts`, around lines 55-78:

After calling `AttendeeService.getAttendeeByAttendeeId(eventId, attendeeIdCode)`, immediately check whether the returned value is null or undefined. If it is, return a 404 response with JSON body `{ error: "Attendee not found" }` instead of continuing execution.

Only access `attendee.id`, `attendee.name`, `attendee.email`, or `attendee.company` after the null check passes.

If `getAttendeeActivitiesDetailed` expects the internal `attendee.id` rather than `attendeeIdCode`, call it with `attendee.id` after the check.

Follow the same error response pattern used in `EventController.getEventById`.

After the change, add a unit test case for the "attendee not found" scenario.
```

## Severity Guidelines

| Severity | When to Use | Example Issues |
| --- | --- | --- |
| Critical | Immediate data loss, security breach, production crash, auth bypass | SQL injection, missing auth on sensitive endpoint, crash in startup path |
| High | Logic bug that breaks a core feature, missing critical error handling, major performance regression | Wrong calculation in pricing engine, payment race condition, N+1 query on hot path |
| Medium | Maintainability issue likely to cause future bugs, missing tests for new logic, minor behavior issue | Duplicate validation logic, magic values, missing edge-case test |
| Low | Explicit style or clarity issue with low risk | Wrong import order when standardized, confusing name in touched code |

Rule of thumb: if you would not block a real merge for it, do not mark it Medium or higher.

## Sample Review Output

````markdown
## High Issue: Missing null check on attendee lookup can cause runtime error

**File:** `src/controllers/EventController.ts:62-75`
**Category:** Bug

**Problem:**
`getAttendeeByAttendeeId` can return null when the attendee does not exist. The code immediately accesses `attendee.id` without checking, causing a potential 500 instead of a proper 404.

**Impact:**
Users will see generic server errors instead of clear "not found" messages, and the system will mask a predictable edge case as an unexpected failure.

**Suggested Fix:**
```diff
 const attendee = await AttendeeService.getAttendeeByAttendeeId(eventId, attendeeIdCode);
+if (!attendee) {
+  return { status: 404, body: JSON.stringify({ error: "Attendee not found" }) };
+}
 const activities = await AttendeeService.getAttendeeActivitiesDetailed(eventId, attendeeIdCode);
```

**Handoff Prompt:**
```
In the file `src/controllers/EventController.ts`, around lines 62-75:

After the line `const attendee = await AttendeeService.getAttendeeByAttendeeId(eventId, attendeeIdCode);`, add an immediate null/undefined check. If attendee is falsy, return a 404 response with JSON body `{ error: "Attendee not found" }`.

Only proceed to call `getAttendeeActivitiesDetailed` and access attendee properties after the check passes.

Use the same error response format as `getEventById`.

Add a corresponding test case for the "attendee not found" path.
```

### PR Summary
This PR adds attendee activity tracking to the event endpoint. The core direction is sound and follows existing patterns, but one High issue should be fixed before merge.

### Recommendations
- [ ] Fix the null-check issue
- [ ] Add coverage for the not-found path

### Statistics
- Files changed: 4
- Issues found: 1 (Critical: 0, High: 1, Medium: 0, Low: 0)
- Checks run: `npm test -- AttendeeService`
````

## GitHub CLI Commands

Create a review with a markdown body:

```bash
gh pr review 123 --comment --body-file /tmp/review.md
```

Post as a single PR comment:

```bash
gh pr comment 123 --body-file /tmp/review.md
```

Create line-specific comments with the API:

```bash
gh api repos/OWNER/REPO/pulls/123/reviews \
  -f event=COMMENT \
  -f body="Overall review summary here" \
  -f comments='[{"path":"src/file.ts","line":42,"body":"Specific comment here"}]'
```

## Pitfalls to Avoid

- Do not flag every formatting issue unless `REVIEW.md` explicitly requires it.
- Do not suggest massive refactors unless the PR itself is a refactor.
- Do not provide vague feedback like "this could be better".
- Do not include stale PR numbers, dates, or environment-specific details in reusable templates.
- If the PR is good, say what is good. Positive signal builds trust.
