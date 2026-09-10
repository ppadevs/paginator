# Implementation Plan

> Scope: research Option A (Fixes 1+2 only). Single-component change inside the paginator; no automated tests exist or will be added — validation is manual and query-log-observable, per design.md. All work is sequential (one shared modification point), so no `(P)` markers apply.

- [x] 1. Count query rework
- [x] 1.1 Strip sorting from the count query
  - Once filtering and ordering are fully applied to the list query, build the count query from a copy of it and remove the sort order from the copy only
  - Leave a source comment marking that the count copy must be created after the order specifications are applied, so future edits preserve list ordering
  - Observable: the generated count statement contains no ORDER BY, while the list statement keeps every mandatory and user order specification
  - _Requirements: 1.1, 1.2_
- [x] 1.2 Execute the count exactly once with explicit grouped handling
  - Branch on whether the query groups rows: grouped queries read all per-group count rows in a single execution and take the number of rows as the total; non-grouped queries take the single scalar result
  - Cast the total to an integer on both paths; derive total pages exactly once after the branch, keeping the existing ceiling behavior
  - Remove the exception-driven fallback execution, the broad error swallowing, and the exception import that becomes unused; count errors propagate to the caller
  - Observable: a grouped screen logs exactly one COUNT statement per page load and reports the same total as before; an empty filtered set reports 0 items and 0 pages on both grouped and non-grouped paths
  - _Depends: 1.1_
  - _Requirements: 2.1, 2.2, 2.3, 3.1, 3.2, 3.3, 4.1, 4.2, 4.3_
- [ ] 2. Validation and release readiness
- [ ] 2.1 Verify totals parity and single execution on consuming screens
  - With the Doctrine SQL logger enabled, replay representative screens: one COUNT per pagination run, no ORDER BY in the count statement, totals and page slices identical to the previous release for both 1:1-join and grouped screens
  - Exercise the edge cases from design.md: an empty filter result yields total 0 and pages 0 with an integer total; a deliberately broken order field raises its exception instead of leaving the total unset
  - Observable: the design.md verification checklist passes completely (query-log, profiler count, replay parity, edge cases)
  - _Depends: 1.1, 1.2_
  - _Requirements: 1.3, 2.3, 3.1, 3.2, 4.1, 4.2, 4.3, 5.2_
- [ ] 2.2 Confirm the unchanged public surface
  - Compare the class against the previous release: construction, pagination entry point, and result getters keep their names and signatures; no method added, removed, or renamed
  - Observable: an API diff against the previous release shows changes only inside the count block; design.md review passes
  - _Depends: 1.2_
  - _Requirements: 5.1_
