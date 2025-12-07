# Empty Stats Card Fix - Design Document

**Date:** 2025-11-21
**Status:** Approved
**Complexity:** Low (1 component change + tests)

## Problem Statement

The `UiDetailQuickStatsCard` component always renders a `UCard` wrapper, even when the `stats` array is empty. This creates visual clutter on detail pages where entities have no quick stats to display.

**Example:** Item "acid-absorbing-tattoo" has no cost or weight, resulting in an empty card appearing between the header and description sections.

## Solution

Add conditional rendering to `UiDetailQuickStatsCard` so it only renders when `stats.length > 0`.

### Design Decision: Component-Level vs. Page-Level Conditional

**Chosen Approach:** Component-level conditional rendering (Approach 1)

**Rationale:**
- **Single source of truth:** Logic lives in one place (the component)
- **Automatic benefit:** All 6 detail pages fixed immediately
- **Future-proof:** New pages automatically handle empty stats correctly
- **Aligns with component responsibility:** The card knows when it shouldn't exist
- **Minimal change:** One-line template modification

**Rejected Alternatives:**
- **Page-level conditionals:** Would require updating 6 pages, repetitive logic, maintenance burden
- **Empty state message:** Adds visual noise instead of removing it, doesn't solve the UX problem

## Implementation Plan

### 1. TDD Process (RED-GREEN-REFACTOR)

**RED Phase - Write Failing Test:**
```typescript
it('does not render when stats array is empty', () => {
  const wrapper = mount(UiDetailQuickStatsCard, {
    props: { stats: [] },
    ...mountOptions
  })
  expect(wrapper.find('.card').exists()).toBe(false)
})
```

**GREEN Phase - Implement Fix:**
```vue
<template>
  <UCard v-if="stats.length > 0">
    <div class="grid grid-cols-1 gap-6" :class="gridClass">
      <!-- existing content -->
    </div>
  </UCard>
</template>
```

**REFACTOR Phase:**
- Verify all existing 8 tests still pass
- Add additional defensive test if needed
- Run full test suite (517 tests)

### 2. Manual Verification

**Test Cases:**
- ✅ `http://localhost:3000/items/acid-absorbing-tattoo` - Should NOT show empty card
- ✅ `http://localhost:3000/items/longsword` - Should show stats card (cost, weight, damage)
- ✅ `http://localhost:3000/spells/fireball` - Should show stats card (casting time, range, etc.)

### 3. Documentation

**CHANGELOG.md:**
```markdown
### Fixed
- Empty stats card no longer renders when no quick stats available (2025-11-21)
```

## Impact Analysis

### Affected Pages
- **Items:** Primary beneficiary (many items lack cost/weight)
- **Races:** Potential benefit (missing size/speed data)
- **Classes, Backgrounds, Feats:** Edge case protection
- **Spells:** No visible change (always have 4 stats)

### Breaking Changes
**None.** Existing behavior preserved for non-empty stats arrays.

### Data Flow

**Before:**
```
Page → stats: [] → Component → Renders empty UCard → User sees empty box
```

**After:**
```
Page → stats: [] → Component → Renders nothing → Clean UI
```

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| `stats: []` | No render ✅ |
| `stats: [one]` | Renders normally ✅ |
| `stats: [many]` | Renders normally ✅ |
| `stats: null/undefined` | TypeScript prevents (type is `Stat[]`) ✅ |

## Testing Strategy

**New Tests:**
1. Component doesn't render with empty stats array
2. Component doesn't render card wrapper when empty

**Existing Tests:**
- All 8 current tests continue passing (they provide non-empty arrays)

**Full Suite:**
- 517 tests should remain green after implementation

## Success Criteria

- [x] Design documented
- [ ] Test written first (RED)
- [ ] Implementation added (GREEN)
- [ ] All tests pass (REFACTOR)
- [ ] Manual browser verification complete
- [ ] CHANGELOG.md updated
- [ ] Work committed to git

## Timeline

**Estimated Effort:** 15 minutes
- 5 min: Write test (RED)
- 2 min: Implement fix (GREEN)
- 5 min: Manual verification
- 3 min: Update CHANGELOG + commit

---

**Next Step:** Proceed to implementation following TDD workflow.
