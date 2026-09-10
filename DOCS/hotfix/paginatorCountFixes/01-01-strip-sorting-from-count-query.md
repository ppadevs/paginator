---
task: "1.1 - Strip sorting from the count query"
spec: "paginatorCountFixes (research Option A)"
branch: "hotfix-paginatorCountFixes"
status: "complete"
---

## Task Overview

**Objective**: The count query built during pagination must never execute ORDER BY sorting. The sort order is removed from the count copy only — the list query keeps every mandatory and user order specification.

**Dependencies**: none (first task; the count copy is created after order-spec handling, which the current code already guarantees structurally)

**Boundary**: Paginator count block (construction of the count query inside the pagination flow)

**Requirements**: 1.1, 1.2

## Implementation Steps

### Step 1: Remove the sort order from the count copy
- [ ] Immediately after the list query builder is copied for counting, reset the ORDER BY part on the copy
- [ ] Do not touch the original list query builder — its order parts stay exactly as applied by the mandatory and user order handling above
- **Observable**: the count statement sent to the database contains no ORDER BY clause

### Step 2: Guard the insertion point with a source comment
- [ ] Add a comment directly at the copy site explaining that the copy must be created **after** the order-spec handling, because removing the order parts only works on the copy
- **Observable**: a future reader sees why the removal happens here and keeps the ordering intact when refactoring

### Step 3: Verify the list query still carries its ordering
- [ ] Inspect the generated list statement: all mandatory order specs and user order specs must appear unchanged
- [ ] Run a paginated screen and confirm rows still come back in the expected order
- **Observable**: list rows are returned in the same order as before the change

## Files to Create/Modify

| File | Action | Purpose |
|------|--------|---------|
| `src/Paginator/Paginator.php` | Modify | Drop the ORDER BY part from the count copy only |

### File: `src/Paginator/Paginator.php`

**Current Code** (lines 263–265, inside `paginate()`; the count copy is created after the mandatory-order block at lines 239–249 and the user-order block at lines 251–261):

```php
		// total items
		$countQb = clone $qb;
		$aliases = $countQb->getAllAliases();
```

**Expected Code After Implementation**:

```php
		// total items
		// the count copy MUST be created after the order-spec handling above:
		// the copy carries the order parts, so resetting them here removes
		// sorting from the count query only - the list query keeps its order
		$countQb = clone $qb;
		$countQb->resetDQLPart('orderBy');
		$aliases = $countQb->getAllAliases();
```

> The execution of the count (try/catch block below these lines) is **not** changed in this task — task 1.2 replaces it.

## Acceptance Criteria

- [ ] The count statement logged by Doctrine contains no ORDER BY (1.1)
- [ ] The list statement logged by Doctrine retains all order specifications (1.2)
- [ ] Only the count copy is modified; the list query builder is untouched
- [ ] All requirements covered: 1.1, 1.2

## Notes

- Placement is the whole point of this task: the removal must happen on the copy, after order handling. Resetting on the list query would break screen ordering; resetting before the order specs are applied would be undone by them.
- Sorting can never change a row count, so removing it from the count is behavior-preserving — totals must stay identical.
