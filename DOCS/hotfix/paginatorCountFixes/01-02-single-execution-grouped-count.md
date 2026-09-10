---
task: "1.2 - Execute the count exactly once with explicit grouped handling"
spec: "paginatorCountFixes (research Option A)"
branch: "hotfix-paginatorCountFixes"
status: "complete"
---

## Task Overview

**Objective**: Replace the exception-driven count execution (which runs the count twice for grouped queries and silently swallows unrelated errors) with a single explicit execution: grouped queries take the number of per-group rows as the total; non-grouped queries take the scalar result. Totals become integers on both paths.

**Dependencies**: 1.1 (sorting already removed from the count copy)

**Boundary**: Paginator count block (execution of the count query and totals bookkeeping)

**Requirements**: 2.1, 2.2, 2.3, 3.1, 3.2, 3.3, 4.1, 4.2, 4.3

## Implementation Steps

### Step 1: Branch on the presence of grouping
- [ ] If the count query groups rows, execute it once as a list and use the number of returned rows as the total (each row is one group)
- [ ] Otherwise, execute it once as a single scalar result
- **Observable**: a grouped screen issues exactly one COUNT statement per pagination run (previously two)

### Step 2: Make totals integers on both paths
- [ ] Cast the total to an integer on the grouped path (row count) and on the non-grouped path (database drivers may return numeric strings)
- **Observable**: the total accessor returns an int after every successful pagination run, grouped or not

### Step 3: Keep the page-total derivation exactly once, after the branch
- [ ] Keep the existing single assignment that derives total pages from the total and the item count (ceiling behavior unchanged)
- **Observable**: an empty filtered set yields total 0 and total pages 0 on grouped and non-grouped paths

### Step 4: Remove the exception flow and the unused import
- [ ] Delete the broad catch block and the now-unused exception import; errors from the count query propagate to the caller
- **Observable**: a deliberately broken order field raises its exception instead of leaving the total unset

## Files to Create/Modify

| File | Action | Purpose |
|------|--------|---------|
| `src/Paginator/Paginator.php` | Modify | Single-execution count branch, integer totals, no swallowed errors |

### File: `src/Paginator/Paginator.php`

**Current Code** (line 5 and lines 266–279; the count copy above is already order-free after task 1.1):

```php
use Doctrine\ORM\NonUniqueResultException;
```

```php
		try{
		    $this->totalItems = $countQb->select("COUNT('{$aliases[0]}')")
		    ->getQuery()
		    ->getSingleScalarResult();
		} catch (\Exception $e){
		    if($e instanceof NonUniqueResultException){
		        $totalItems = $countQb->select("COUNT('{$aliases[0]}')")
		        ->getQuery()
		        ->getScalarResult();
		        end($totalItems);
		        $this->totalItems = key($totalItems)+1;
		    }
		}
		$this->totalPages = ceil($this->totalItems / $this->request->getItemCount());
```

**Expected Code After Implementation**:

```php
		if (!empty($countQb->getDQLPart('groupBy'))) {
		    // the query returns one row per group; total items == number of groups
		    $rows = $countQb->select("COUNT('{$aliases[0]}')")
		    ->getQuery()
		    ->getScalarResult();
		    $this->totalItems = (int) \count($rows);
		} else {
		    $this->totalItems = (int) $countQb->select("COUNT('{$aliases[0]}')")
		    ->getQuery()
		    ->getSingleScalarResult();
		}
		$this->totalPages = ceil($this->totalItems / $this->request->getItemCount());
```

**Expected Code After Implementation** (line 5 — import removed):

```php
use Doctrine\ORM\QueryBuilder;
use Doctrine\ORM\Query;
```

> The `COUNT('…')` expression style and the root-alias assumption are intentionally unchanged (out of scope). The old grouped fallback `end(); key()+1` equals the row count for the 0-indexed list, so totals are identical — the query just runs once instead of twice.

## Acceptance Criteria

- [ ] Grouped queries execute exactly one count query per pagination run (2.1)
- [ ] Grouped totals equal the number of groups, identical to the previous release (2.2, 2.3)
- [ ] Empty filtered sets produce total 0 / pages 0, including the grouped case that previously left the total unset (3.1, 3.2)
- [ ] Count errors propagate to the caller instead of being swallowed (3.3)
- [ ] The total is an integer on both paths; pages derive once from total × item count; 0 items → 0 pages (4.1, 4.2, 4.3)
- [ ] All requirements covered: 2.1, 2.2, 2.3, 3.1, 3.2, 3.3, 4.1, 4.2, 4.3

## Notes

- The old catch only handled the multi-row case and left every other error silent with an unset total — the new code has no catch at all, so the empty-grouped hole (no rows → no result → previously unhandled exception type) is closed as a side effect.
- Do not "improve" the quoted `COUNT('alias')` style here; changing it is explicitly out of scope.
