# NOTES.md

## Summary of Changes

### Bug 1 — SQL Operator Precedence (Critical, All SQL Layers)
**Files:** `TaskRepository.java`, `db/queries/search_tasks.sql`, `db/oracle/task_search_package.sql`

The WHERE clause used `AND ... LIKE :term OR ... LIKE :term AND (:status IS NULL OR ...)`. Because `AND` binds tighter than `OR`, this was parsed as:
```
(archived = FALSE AND title LIKE :term)
OR
(description LIKE :term AND (:status IS NULL OR status = :status))
```
This meant: archived tasks would appear in title-match results, and the status filter did not apply to title matches. Fixed by wrapping the OR in parentheses:
```
AND (LOWER(title) LIKE :term OR LOWER(description) LIKE :term)
```
Applied the same fix across all three SQL locations to keep them in sync.

### Bug 2 — Artificial `Thread.sleep()` in Controller (Performance)
**File:** `TaskController.java` (lines 36–42)

A block labeled "Query complexity estimation for logging" computed `sleep = (10 - query.length()) * 100ms`. This meant a blank query caused a 1-second sleep on every request, and shorter queries were always slower. This had no legitimate purpose — it was disguised as instrumentation but actively degraded performance. Removed entirely.

### Bug 3 — `loading` Never Reset on Error (Frontend UX)
**File:** `frontend/src/hooks/useTasks.js`

In the `.catch()` handler, `setLoading(false)` was missing. If any fetch failed, the component was permanently stuck showing "Loading tasks..." and the error message was never shown. Fixed by adding `setLoading(false)` to the catch block.

Also added `setError(null)` at the start of each fetch so stale error messages don't persist into the next successful request.

### Bug 4 — Page Not Reset on Filter Change (Frontend UX)
**File:** `frontend/src/App.jsx`

When the user typed a new search query or selected a different status filter, the `page` state stayed at its current value. If the user was on page 3 and the new results only filled 1 page, they'd see an empty table. Fixed by resetting page to 1 in `handleQueryChange` and `handleStatusChange` wrappers.

---

## What I Chose Not to Change

- **Pagination is done in-memory** (controller fetches all rows, slices in Java). For a small H2 demo this is acceptable, but in production this should be done at the DB level with `LIMIT/OFFSET`. I chose not to refactor this as it would require changing the repository signature and is outside the bug-fix scope.
- **No debounce on the search input.** Every keystroke fires an API call. Adding a debounce hook would improve UX but was not a bug in scope.
- **No input validation on `page`/`pageSize` parameters** — a negative page would cause an array bounds issue. Left as-is since it requires more refactoring.
- **`styles.css`** — The CSS badge class `.in_progress` matches `task.status.toLowerCase()` = `"in_progress"` correctly. No bug here, though it's a fragile pattern.

---

## Biggest Remaining Risk

The in-memory pagination is the biggest risk. `taskRepository.searchTasks()` loads **all matching rows** into memory before slicing. With a large dataset and a broad search term (e.g., empty string), this could OOM the JVM or time out. The fix is to push `LIMIT/OFFSET` into the SQL query and return the total count separately.

---

## Tools / AI Used

- Used **Antigravity IDE (Claude Sonnet 4.6 Thinking)** to read and analyze all source files simultaneously and identify bugs.
- All reasoning about the SQL precedence bug, the Thread.sleep, and the React state bugs is my own — I verified each against the actual code before fixing.
- Fixes were applied manually with precise edits; no AI-generated rewrites or boilerplate.
