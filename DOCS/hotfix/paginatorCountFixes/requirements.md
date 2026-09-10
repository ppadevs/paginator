# Requirements Document

## Introduction
This spec delivers research Option A for the in-house pagination package: two behavior-preserving performance fixes to the row-count query built during pagination. Row counting shall never execute ORDER BY sorting, and screens whose queries group rows shall execute the count exactly once instead of twice. Totals stay identical to the previous release for screens whose joins do not multiply rows, and no consuming call site needs to change. Out of scope behavior (count caching, distinct counting) remains available for later specs.

## Boundary Context
- **In scope**: How the count query is constructed and executed inside pagination — absence of sort clauses in the count, single count execution for grouped and non-grouped queries, count result handling for empty and failing counts, integer totals, and unchanged list-query behavior.
- **Out of scope**: Count result caching (research Fix 3), distinct counting for row-multiplying joins (research Fix 4), reducing or conditionally skipping filter joins (application-side optimization, crash doc 10), changing the existing count expression style, and any change to how list rows are fetched or hydrated.
- **Adjacent expectations**: Consuming applications continue calling the package unchanged and upgrade via their package manager; verification is operator-observable through the database query log and by comparing totals with the previous package release.

## Requirements

### Requirement 1: Count query independent of sort order
**Objective:** As a screen developer, I want row counting to ignore sort order, so that count queries never pay for sorting they cannot use.

#### Acceptance Criteria
1. When a paginated request is processed, the Paginator shall build its count query without any ORDER BY clause, as observable in the database query log.
2. When a paginated request includes mandatory and user order specifications, the Paginator shall apply all of them to the paginated list query unchanged.
3. When the same request is replayed on the previous release and on the fixed release, the Paginator shall return the same total items value for queries whose joins do not multiply rows.

### Requirement 2: Single count execution for grouped queries
**Objective:** As a screen developer, I want screens with grouped queries to execute the count exactly once, so that page latency drops by the cost of the duplicated count (e.g. ~14 s on the picking-plan screen).

#### Acceptance Criteria
1. When the query contains GROUP BY, the Paginator shall execute exactly one count query per pagination run, as observable in the database query log.
2. When the query contains GROUP BY, the Paginator shall set total items to the number of groups returned by the grouped count.
3. When the same grouped request is replayed on the previous release and on the fixed release, the Paginator shall return the same total items value (the number of groups).

### Requirement 3: Predictable totals for empty and failing counts
**Objective:** As a screen developer, I want predictable totals when no rows match or the count fails, so that screens never silently show an unset-total state caused by swallowed errors.

#### Acceptance Criteria
1. When the filtered result set is empty, the Paginator shall set total items to 0 and total pages to 0.
2. When the filtered result set is empty and the query contains GROUP BY, the Paginator shall set total items to 0 instead of leaving the total unset.
3. If count execution raises an unexpected error, the Paginator shall surface that error to the caller instead of silently leaving the total unset.

### Requirement 4: Integer totals on all success paths
**Objective:** As a screen developer, I want totals returned as integers after pagination, so that views and API responses can rely on them.

#### Acceptance Criteria
1. When pagination completes successfully, the Paginator shall return total items as an integer, for grouped and non-grouped queries alike.
2. When pagination completes successfully, the Paginator shall derive total pages from the total items value and the configured item count.
3. When total items is 0, the Paginator shall return total pages of 0.

### Requirement 5: Upgrade without caller changes
**Objective:** As a consuming application integrator, I want the fix delivered without public API changes, so that upgrading the package requires no call-site modifications.

#### Acceptance Criteria
1. The Paginator shall keep its existing public surface (construction, pagination entry point, and result getters) unchanged in this fix.
2. When an existing screen runs unmodified against the fixed package, the Paginator shall return the same page slice and the same row order as the previous release.
