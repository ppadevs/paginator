---
task: "2.2 - Confirm the unchanged public surface"
spec: "paginatorCountFixes (research Option A)"
branch: "hotfix-paginatorCountFixes"
status: "draft"
---

## Task Overview

**Objective**: Prove the fix is purely internal: the class's public surface (construction, pagination entry point, result getters) is byte-for-byte unchanged, and the only source difference versus the previous release lives inside the count block.

**Dependencies**: 1.2 (count rework complete)

**Boundary**: Whole-class API review — no source changes unless the diff reveals an accidental public-surface change (which would be a defect to fix in tasks 1.1/1.2)

**Requirements**: 5.1

## Implementation Steps

### Step 1: API diff against the previous release
- [ ] Diff the class file against the previous released version
- [ ] Confirm: constructor signature unchanged; pagination entry point signature unchanged; all result getters (page number, page totals, item totals, separator, paginated result) unchanged in name and signature; no method added, removed, renamed, or made required
- **Observable**: the diff shows changes only inside the count block (plus the removed unused import) — nothing in any public member

### Step 2: Internal review against the design
- [ ] Walk the diff against design.md: order removal on the copy, single-execution branch, integer casts, import removal, placement comment — nothing else
- [ ] Confirm the empty-grouped behavior change (0 instead of unset) is documented in the release notes draft
- **Observable**: diff maps 1:1 to design decisions; no unrelated changes slipped in

## Files to Create/Modify

| File | Action | Purpose |
|------|--------|---------|
| — | — | Review task: no repository files change |

## Acceptance Criteria

- [ ] Public API unchanged: construction, pagination entry point, and result getters keep names and signatures (5.1)
- [ ] Diff contains only the count-block changes and the unused-import removal
- [ ] Empty-grouped behavior change (0 instead of unset) noted for the release notes
- [ ] All requirements covered: 5.1

## Notes

- Callers must not need any change to adopt the fixed package (5.1) — this diff review is the proof.
- If anything outside the count block appears in the diff, remove it; scope discipline is a hard requirement for this hotfix.
