---
title: "Research & Design Decisions - hotfix/paginatorCountFixes"
date: "2026-09-08"
type: "Hotfix research (Path B)"
source: "11-ppadevs-paginator-fixes.md (doc 11, external crash-analysis docs), backed by slow-log/EXPLAIN analysis in doc 09/10"
target: "src/Paginator/Paginator.php - method paginate()"
proposed_branch: "hotfix-paginatorCountFixes (from master)"
status: "research complete - code claims verified against HEAD caef3c6"
---

# Research & Design Decisions — hotfix/paginatorCountFixes

## Summary
- **Feature**: `orderBy` — performance fixes for the COUNT query inside `ppadevs/paginator` (drop ORDER BY from the count clone, single count execution, opt-in count cache, documented 1:n count semantics).
- **Discovery Scope**: Hotfix (Path B) — bug/performance fix in an in-house package used by hundreds of paginated screens in `PPA_backend`.
- **Key Findings**:
  - All four proposed fixes are technically sound and verified against the current source; Fix 1 and Fix 2 are safe defaults, Fix 3 is opt-in by design, Fix 4 must not be applied silently.
  - **Dependency gap**: `composer.json` requires only `php >=7.4` + `doctrine/orm ^2.11`. `psr/simple-cache` is **not** shipped by doctrine/orm (doc 11's "Doctrine already ships it" refers to the consuming app's `symfony/cache`). Fix 3 requires adding `psr/simple-cache` to this package, with constraint `^1.0 || ^2.0 || ^3.0` to avoid conflicts with host apps on newer symfony/cache.
  - **Discovered edge case** (not in doc 11): a grouped count over zero matching rows throws `NoResultException`, which the current broad `catch (\Exception)` does not handle → `totalItems` stays `null`. Fix 2's restructure incidentally fixes this (`\count([]) = 0`).
  - The repo has **no test suite** (no phpunit, no `tests/`); the doc-11 regression checklist is the only safety net — worth adding a minimal PHPUnit suite with this hotfix.
  - Doc 11's line references are accurate for HEAD: count block at `Paginator.php:263-279`, order-spec handling at `:239-261`, clone created *after* order handling (so Fix 1's placement constraint already holds structurally).

## Research Log

### Source document & problem context
- **Context**: Dashboard screens are slow (doc 09/10); the COUNT queries behind pagination are generated in `Paginator::paginate()`. Because the package is in-house (`github.com/ppadevs/paginator`), one release fixes every screen.
- **Sources Consulted**: doc 11 (`11-ppadevs-paginator-fixes.md`), doc 09 (slowlog script) / doc 10 (EXPLAIN, conditional joins) referenced therein, this repo's source at HEAD.
- **Findings**: All proposed changes target `paginate()` in `src/Paginator/Paginator.php`. Three concrete problems confirmed in code:
  1. `$countQb = clone $qb;` (`Paginator.php:264`) inherits joins, WHERE, GROUP BY **and ORDER BY** — ORDER BY can never change a row count but is executed (potential filesort; and under `ONLY_FULL_GROUP_BY` it can even break the grouped count query).
  2. GROUP BY queries run the count **twice**: `getSingleScalarResult()` throws `NonUniqueResultException` (one row per group), and the catch re-executes the identical query via `getScalarResult()`. The picking-plan screen pays ~2 × 14 s per page.
  3. The catch is `catch (\Exception $e)` and only handles `NonUniqueResultException`; every other exception is swallowed → `$this->totalItems` stays `null` → `ceil(null/itemCount)` = 0 pages, silently.
- **Implications**: Fixes belong in the package; caller-visible behavior must stay backwards-compatible (totals identical for 1:1-join screens, grouped totals = number of groups).

### Current implementation verification (lines 263–279)
- **Context**: doc 11 quotes the count block as "lines ~264–279"; verify exact anchors for implementation.
- **Findings**:
  - `// total items` comment: line 263; clone + count: lines 264–278; `totalPages`: line 279. Doc is accurate.
  - Mandatory order specs: lines 239–249; user order specs: lines 251–261; the count clone (line 264) is created **after** both — doc 11's requirement "reset must come after order-spec handling" is already satisfied by inserting `resetDQLPart('orderBy')` immediately after the clone.
  - `COUNT('{$aliases[0]}')` uses a quoted alias (DQL string literal → SQL `COUNT('alias')`, which counts non-NULL constants ≡ `COUNT(*)`). Quirky but existing, working behavior observed in the slow log; **out of scope** — do not change alongside these fixes.
  - `getAllAliases()` returns the root alias at index 0 — pre-existing assumption, unchanged by the fixes.
- **Impagination**: None — anchors confirmed; implementations can cite exact line numbers.

### Fix 1 — drop ORDER BY from the count clone (verified safe)
- **Context**: Doc 11 claims `resetDQLPart('orderBy')` on the clone is always safe and must not touch the list query.
- **Sources Consulted**: `src/Paginator/Paginator.php:201-288`; Doctrine ORM ^2.11 `QueryBuilder` API.
- **Findings**:
  - `QueryBuilder::resetDQLPart('orderBy')` exists in all supported doctrine/orm ^2.11 versions; resetting on the clone only affects the count.
  - The paginated list query (`$qb`, lines 282–285) is a *different* object — ordering untouched. ✅
  - Zero behavioral change to totals (ORDER BY cannot affect a row count); it also removes a potential filesort and a class of `ONLY_FULL_GROUP_BY` errors on grouped counts.
- **Implications**: Can ship as unconditional default behavior; no caller changes; no opt-in needed.

### Fix 2 — grouped counts: single execution, no exception flow (verified equivalent)
- **Context**: Doc 11 replaces the try/catch double-execution with an explicit branch on `getDQLPart('groupBy')`.
- **Sources Consulted**: `Paginator.php:263-279`; Doctrine `AbstractQuery::getSingleScalarResult()/getScalarResult()` semantics.
- **Findings**:
  - Double execution confirmed: `getSingleScalarResult()` → `NonUniqueResultException` → catch re-runs `getScalarResult()` on the same query. ~2 × count cost on every grouped screen.
  - Equivalence holds: `getScalarResult()` returns a **0-indexed** array, so `end($rows); key($rows)+1` ≡ `\count($rows)`. New code: `totalItems = \count($rows)` where each row is the per-group `COUNT`. Same number, half the executions.
  - **Edge case discovered**: zero matching rows + GROUP BY → SQL returns no rows → `getSingleScalarResult()` throws `NoResultException` (not `NonUniqueResultException`) → today's code leaves `totalItems = null` (silent 0 pages). The Fix 2 branch returns `\count([]) = 0` — strictly better, but confirm no caller depends on `getTotalItems()` returning `null`.
  - With the catch removed, the `use Doctrine\ORM\NonUniqueResultException;` import (line 5) becomes unused → remove.
  - `totalPages` (line 279) must be computed exactly once, after the new branch (doc 11's snippets each show it; keep a single assignment).
  - **Implications**: Non-grouped path keeps `getSingleScalarResult()` (returns exactly one row for a plain COUNT — no catch needed); grouped path uses `getScalarResult()` + `\count()`. Consider casting to `int` on all assignment paths (MySQL/PDO can return numeric strings for COUNT today).

### Fix 3 — opt-in count cache (feasible; one dependency gap found)
- **Context**: Doc 11 proposes `setCountCache(Psr\SimpleCache\CacheInterface, ttl)` with SQL+params cache key, TTL 60 s, off by default.
- **Sources Consulted**: `composer.json` (this repo), doc 11 deployment notes, PSR-16.
- **Findings**:
  - **Gap**: doc 11 claims "Doctrine already ships `psr/simple-cache`" — true for the *app* (`symfony/cache`), false for *this package*: `composer.json` requires only `php >=7.4` and `doctrine/orm ^2.11`; `psr/simple-cache` is not a doctrine/orm dependency (doctrine/cache is merely suggested). The package must add it, otherwise the type-hint fatals on bare installs.
  - **Version gotcha**: newer `symfony/cache` (6.4+/7.x in the app) pulls `psr/simple-cache ^2.0 || ^3.0`; pinning `^1.0` would conflict. Use `"psr/simple-cache": "^1.0 || ^2.0 || ^3.0"` — PSR-16's interface is stable for our `get()/set()` usage across all three; 1.0 keeps PHP 7.4 installs resolvable.
  - **Cache key**: `md5($countQuery->getSQL() . serialize($countQuery->getParameters()))` is deterministic and precise — `getParameters()` is an `ArrayCollection` of `Doctrine\ORM\Query\Parameter` objects whose serialized state is stable; SQL + bound params is the right granularity (DQL-part hashing would be unsafe, as doc 11 notes).
  - **Hit/miss flow**: `$this->totalItems` starts `null`, so the `isset` guard works; `$cached !== null` correctly treats `0` as a hit; cast `(int)` on write-back so `getTotalItems()` returns int on both hit and miss (regression item in doc 11).
  - TTL 60 s is a sane dashboard default; totals may be up to TTL stale — acceptable, and the reason this must stay **opt-in** (no caller changes until a screen calls `setCountCache()`).
- **Implications**: Fix 3 is additive API (new optional setter + two properties); default behavior unchanged; can ship in the same release as Fixes 1+2.

### Fix 4 — 1:n join count semantics (verified: flag does not exist yet)
- **Context**: Doc 11 documents (does not fix silently) that `COUNT('root-alias')` counts multiplied rows when a 1:n join is present; today's totals are correct only because current joins are 1:1.
- **Sources Consulted**: `src/Paginator/AbstractPaginatedQueryRequest.php`.
- **Findings**:
  - No `distinctCount` flag exists — confirmed. Adding it means: protected property (default `false`), getter/setter, and an optional constructor param (default) to preserve the 5–7-arg constructor BC.
  - Per-site opt-in only: `COUNT(DISTINCT root.id)` when the flag is set. Never enable globally.
- **Implications**: Out of the immediate hotfix unless a call site needs it; document as opt-in behavior in the release notes.

### Repo constraints
- **Context**: Feasibility of landing the fix in this package.
- **Sources Consulted**: `composer.json`, repo tree, git state.
- **Findings**:
  - `php >=7.4` → all proposed code is 7.4-compatible (no enums/readonly/promoted properties needed).
  - `doctrine/orm ^2.11` → `resetDQLPart`, `getDQLPart`, `getAllAliases`, `getScalarResult` all available.
  - **No tests**: no `tests/` directory, no phpunit in composer.json. Regression safety = doc 11's manual checklist (profiler query counts, production comparisons). Adding a small PHPUnit suite (non-grouped count, grouped count single-execution, orderBy stripped from count query, cache hit/miss) is strongly recommended with this hotfix.
  - `README.md` is a stub; no CHANGELOG → add a CHANGELOG entry + docblock for `setCountCache()`.
  - Working tree clean, `master` in sync with `origin/master` at `caef3c6` — clean base for `hotfix-paginatorCountFixes`.
- **Implications**: Low-risk package change; the structural join work stays in the app repo (doc 10), as doc 11 itself states — the package fix does **not** remove filter-join walks.

## Architecture Pattern Evaluation

| Option | Description | Strengths | Risks / Limitations | Notes |
|--------|-------------|-----------|---------------------|-------|
| A. Fixes 1+2 only | Release resetDQLPart + single-execution branch as new default behavior | Zero new deps, minimal diff, immediate win on every grouped screen | No cache; heavy screens still pay one count | Smallest safe hotfix |
| B. Fixes 1+2 + Fix 3 opt-in cache (recommended) | Same as A plus optional `setCountCache()` API and `psr/simple-cache` dep | ~0 s counts on cache hits for opting screens; zero caller changes until opt-in | New dependency; stale totals ≤ TTL; cache-key code to maintain | One release improves everything; app opts in per screen |
| C. Fixes 1+2+3+4 together | Include `distinctCount` flag in same release | Complete package story | Fix 4 changes totals if ever defaulted; more surface to review with no tests | Rejected for hotfix — Fix 4 is documented, opt-in only, separate task |
| D. Vendor patch stopgap | `cweagans/composer-patches` with the diff until a tagged release | Unblocks app immediately | Divergence risk, extra tooling in app | Fallback per doc 11 deployment §2, not the primary path |

## Design Decisions

### Decision: ship Fix 1 + Fix 2 as default behavior; Fix 3 as opt-in API; Fix 4 documentation-only
- **Context**: Doc 11 defines three safe-by-default changes and two opt-in changes; hotfix must be backwards-compatible for hundreds of call sites.
- **Alternatives Considered**:
  1. Everything in one release (Option C)
  2. Minimal release (Option A) with cache later
  3. Fixes 1+2 + opt-in cache now (Option B)
- **Selected Approach**: Option B — `resetDQLPart('orderBy')` after the count clone; explicit `groupBy` branch replacing the exception flow; `setCountCache()` behind `psr/simple-cache`, default off.
- **Rationale**: Fixes 1+2 are provably semantics-preserving (verified above); Fix 3 is additive and inert until opted in; Fix 4 alters totals and is explicitly "do not change silently" in doc 11.
- **Trade-offs**: Slightly larger diff than pure 1+2; +1 dependency. Buys the ~99 % count-execution reduction on opted-in dashboards without touching any caller.
- **Follow-up**: Verify during implementation that (a) the list query retains its orderBy, (b) grouped screens issue exactly one count query, (c) `getTotalItems()` returns int on cache hit and miss.

### Decision: replace the broad `catch (\Exception)` with the explicit groupBy branch (and fix the empty-result hole)
- **Context**: Current code swallows every non-`NonUniqueResultException` error, leaving `totalItems = null` (also broken for `NoResultException` on empty grouped results).
- **Alternatives Considered**:
  1. Keep the try/catch but narrow it to `NonUniqueResultException` only
  2. Branch on `getDQLPart('groupBy')` (doc 11's Fix 2), no catch on the non-grouped path
- **Selected Approach**: Option 2; `getSingleScalarResult()` on a non-grouped COUNT always yields exactly one row, so no exception flow is needed.
- **Rationale**: Deterministic control flow, half the executions, no swallowed exceptions; empty grouped result now cleanly yields `totalItems = 0`.
- **Trade-offs**: Behavioral delta in the previously-broken empty case (0 vs null) — an improvement, but must be noted in the release notes.
- **Follow-up**: Search `PPA_backend` for callers relying on `getTotalItems() === null` before release (expected: none).

## Synthesis (spec-design, 2026-09-08)
- **Scope decision**: Option A (Fixes 1+2 only) was selected for this spec, superseding the earlier Option B recommendation. Fix 3 (count cache + `psr/simple-cache`) and Fix 4 (`distinctCount`) are deferred to a later spec; the corresponding decisions above are struck from this spec's scope.
- **Generalizations**: None — single-class package, no generalization beyond `paginate()`.
- **Build-vs-adopt**: No new dependencies; only doctrine/orm ^2.11 APIs already in use (`resetDQLPart`, `getDQLPart`, `getScalarResult`).
- **Simplifications**: Removing the exception-driven flow and the unused `NonUniqueResultException` import simplifies the count block from try/catch to a two-branch if/else.
- **Verification direction**: The repo has no test infrastructure and none will be added — acceptance criteria are verified manually via the Doctrine SQL logger, replay comparisons against the previous release, and the slow-log analysis scripts.

## Risks & Mitigations
- **Cache staleness (Fix 3)** — totals up to TTL (60 s) old → opt-in per screen, dashboard-appropriate TTL, documented in the method docblock.
- **psr/simple-cache version conflict with host app** — tri-state constraint `^1.0 || ^2.0 || ^3.0`; verify resolution on PHP 7.4 and PHP 8.x.
- **Totals regression on 1:1-join screens** — Fixes 1+2 are count-equivalent, but compare a few pages against production numbers per doc 11's regression checklist.
- **Silent failure mode today (null totalItems)** is replaced by explicit 0 — confirm no consumer branches on null.
- **No automated tests in this repo** — add minimal PHPUnit coverage for: orderBy removed from count, single count execution on grouped queries, cache hit/miss paths, totalPages math; until then rely on the doc 11 checklist via the Doctrine SQL logger / profiler query count.
- **Release dependency (Fix 3)** — if the app upgrade is delayed, doc 11 §2 (`cweagans/composer-patches`) is the documented stopgap.

## References
- doc 11 (`11-ppadevs-paginator-fixes.md`) — source of the proposed fixes
- doc 09 (`analyze-slowlog.ps1`) and doc 10 (EXPLAIN, conditional joins) — evidence and the structural app-side fix
- `src/Paginator/Paginator.php:201-288` — `paginate()` under fix (count block `:263-279`)
- `src/Paginator/AbstractPaginatedQueryRequest.php` — future home of the opt-in `distinctCount` flag (Fix 4)
- [PSR-16: Common Interface for Caching Libraries](https://www.php-fig.org/psr/psr-16/) — `CacheInterface` contract for Fix 3
- [Doctrine ORM QueryBuilder](https://www.doctrine-project.org/projects/doctrine-orm/en/2.11/reference/query-builder.html) — `resetDQLPart`/`getDQLPart` API
- `.kilo/settings/templates/specs/research.md` — template governing this document

## Proposed Next Steps
1. Create branch `hotfix-paginatorCountFixes` from `master`.
2. `/spec-requirements paginatorCountFixes` → design/tasks for Option A (spec: `DOCS/hotfix/paginatorCountFixes/` — `spec.json`, `requirements.md`).
3. After release: update `ppadevs/paginator` in `PPA_backend` (`composer update ppadevs/paginator`), re-run `scripts\analyze-slowlog.ps1` and `explain-after.sql` to confirm the `COUNT` entries collapse. (Fix 3's `setCountCache()` opt-in is out of scope for Option A and deferred to a follow-up spec.)
