---
task: "2.1 - Verify totals parity and single execution on consuming screens"
spec: "paginatorCountFixes (research Option A)"
branch: "hotfix-paginatorCountFixes"
status: "approved"
---

## Task Overview

**Objective**: Manually verify the implemented count rework against the design.md verification checklist — on a consuming application screen with the Doctrine SQL logger enabled — before release. No automated tests exist in this package and none will be added.

**Dependencies**: 1.1, 1.2 (both count changes implemented)

**Boundary**: Verification only — no package source changes; runs against a consuming application screen (WorkPerformance list or picking-plan) with the updated package installed

**Requirements**: 1.3, 2.3, 3.1, 3.2, 4.1, 4.2, 4.3, 5.2

## Implementation Steps

### Step 1: Query-log check — no sorting in the count, list keeps its order
- [ ] Enable the Doctrine SQL logger / profiler on a consuming screen
- [ ] Load a page and inspect the logged statements: the COUNT statement must contain no ORDER BY; the list statement must retain the full order specification
- **Observable**: logged COUNT without ORDER BY next to a fully ordered list statement

### Step 2: Single execution check on a grouped screen
- [ ] Load a page of a grouped screen (picking-plan) and count the COUNT statements in the profiler
- [ ] Compare with the previous package release: two count executions before, exactly one after
- **Observable**: profiler shows exactly one COUNT statement per page load

### Step 3: Totals parity replay
- [ ] Replay representative requests (1:1-join screens and the grouped screen) against the previous and the fixed release with identical filters and page settings
- [ ] Compare reported totals and page slices row by row
- **Observable**: identical totals and identical page contents between releases

### Step 4: Edge-case checks
- [ ] Apply a filter matching nothing: total must be 0 and pages 0 — on a non-grouped screen and on a grouped screen (the grouped case previously left the total unset)
- [ ] Confirm the total accessor returns an integer in both cases
- [ ] Temporarily break an order field so the count fails: the exception must surface instead of silently showing an unset total
- **Observable**: 0/0 totals as integers on both paths; the broken-order exception propagates to the error handler

## Files to Create/Modify

| File | Action | Purpose |
|------|--------|---------|
| — | — | Verification task: no repository files change; run against a consuming application with the fixed package installed |

## Acceptance Criteria

- [ ] Count statement has no ORDER BY while list statement keeps its order (1.1, 1.2 — regression re-check)
- [ ] Grouped screens show one COUNT per run and totals equal to the previous release (2.1, 2.2 regression, 2.3)
- [ ] Non-multiplying screens show identical totals to the previous release (1.3)
- [ ] Empty filters: total 0 / pages 0, integer type, grouped and non-grouped (3.1, 3.2, 4.1, 4.3)
- [ ] Page totals derive from total × item count on the checked screens (4.2)
- [ ] Count errors surface instead of leaving the total unset (3.3)
- [ ] Page slices identical to the previous release (5.2)
- [ ] All requirements covered: 1.3, 2.3, 3.1, 3.2, 4.1, 4.2, 4.3, 5.2

## Notes

- This is the design.md "Verification" checklist executed for real; it is the only safety net since the package has no test suite — do not skip the parity replay.
- If any parity check fails, stop and return to task 1.2 — do not paper over a totals difference.
- After verification, re-run the slow-log analysis in the consuming app; COUNT entries must have lost their duplicate executions and filesort.
