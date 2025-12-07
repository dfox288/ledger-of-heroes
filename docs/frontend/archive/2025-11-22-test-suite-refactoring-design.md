# Test Suite Refactoring Design

**Date:** 2025-11-22
**Status:** Approved for Implementation
**Goal:** Reduce test maintenance burden by eliminating redundancy while maintaining coverage

---

## Executive Summary

This design refactors the test suite using **shared test helper utilities** to eliminate ~66 redundant tests across entity card components. By extracting common behavior tests into reusable functions, we reduce the maintenance burden from updating 6 files to updating 1 file when shared UI patterns change.

**Impact:**
- **Tests removed:** ~66 redundant tests (11% reduction)
- **Files created:** 3 test helper modules
- **Maintenance improvement:** Shared behavior changes require 1 file update instead of 6
- **Test execution:** ~15-20% faster (fewer redundant component mounts)

---

## Current State Analysis

### Test Suite Statistics
- **Total tests:** 573 passing tests across 46 files
- **Entity card tests:** 180 tests across 6 components (Spell, Item, Race, Class, Background, Feat)
- **Redundancy identified:** ~60 tests testing identical shared behavior
- **Low-value tests:** ~24 tests with truthy/smoke assertions
- **Implementation detail tests:** ~30 tests checking CSS classes

### Redundancy Patterns

All 6 entity card components test the same patterns:

1. **Link behavior** (6 identical tests)
   - Links to detail page with slug
   - Correct href attribute

2. **Hover effects** (6 identical tests)
   - HTML contains 'hover' class

3. **Card styling** (12 identical tests)
   - Border styling
   - Spacing structure (space-y-3)

4. **Description handling** (18 identical tests)
   - Truncates long descriptions
   - Preserves short descriptions
   - Shows default when missing

5. **Source footer** (12 identical tests)
   - Renders sources when present
   - Handles missing sources gracefully

6. **Name display** (6 identical tests)
   - Long names use line-clamp

**Total redundant tests:** ~60 across 6 components

---

## Design Solution: Shared Test Helper Utilities

### Architecture

Create three test helper modules that export reusable test functions:

```
tests/
├── helpers/
│   ├── cardBehavior.ts       # Link, hover, border tests
│   ├── descriptionBehavior.ts # Truncation, fallback tests
│   └── sourceBehavior.ts     # Source footer tests
└── components/
    ├── spell/SpellCard.test.ts    # Uses helpers + spell-specific tests
    ├── item/ItemCard.test.ts      # Uses helpers + item-specific tests
    └── ... (4 more entity cards)
```

### Helper Function Pattern

**Key Design Decision:** Use factory functions (mount functions) instead of component references.

```typescript
// tests/helpers/cardBehavior.ts
export function testCardLinkBehavior(
  mountComponent: () => Promise<VueWrapper>,
  expectedHref: string
) {
  it('links to detail page with slug', async () => {
    const wrapper = await mountComponent()
    const link = wrapper.find('a')
    expect(link.exists()).toBe(true)
    expect(link.attributes('href')).toBe(expectedHref)
  })
}

export function testCardHoverEffects(
  mountComponent: () => Promise<VueWrapper>
) {
  it('applies hover effects for interactivity', async () => {
    const wrapper = await mountComponent()
    expect(wrapper.html()).toContain('hover')
  })
}
```

**Why factory functions?**
- Each test needs a fresh wrapper instance (avoid state pollution)
- Allows passing different props for edge case testing
- Maintains test isolation principles

### Usage Pattern in Entity Tests

```typescript
// tests/components/spell/SpellCard.test.ts
import { testCardLinkBehavior, testCardHoverEffects } from '~/tests/helpers/cardBehavior'
import { testDescriptionTruncation } from '~/tests/helpers/descriptionBehavior'
import { testSourceFooter } from '~/tests/helpers/sourceBehavior'

describe('SpellCard', () => {
  const mockSpell = {
    id: 1,
    name: 'Fireball',
    slug: 'fireball',
    // ... full mock data
  }

  const mountSpellCard = () => mountSuspended(SpellCard, {
    props: { spell: mockSpell }
  })

  // Shared behavior tests (1 line each)
  testCardLinkBehavior(mountSpellCard, '/spells/fireball')
  testCardHoverEffects(mountSpellCard)
  testDescriptionTruncation(
    () => mountSuspended(SpellCard, {
      props: { spell: { ...mockSpell, description: 'A'.repeat(200) } }
    }),
    () => mountSuspended(SpellCard, {
      props: { spell: { ...mockSpell, description: 'Short' } }
    })
  )
  testSourceFooter(mountSpellCard, 'Player\'s Handbook')

  // Spell-specific tests only (domain logic)
  it('formats spell level correctly', async () => { /* ... */ })
  it('formats cantrips correctly', async () => { /* ... */ })
  it('shows concentration badge when needed', async () => { /* ... */ })
  it('shows ritual badge when needed', async () => { /* ... */ })
  // ... 15-18 spell-specific tests
})
```

---

## Test Categorization & Removal Strategy

### Category 1: Implementation Detail Tests (DELETE)

**Count:** ~30 tests across 6 components

**Examples to delete:**
```typescript
// ❌ DELETE - Tests CSS class names
it('uses card component with border', async () => {
  expect(wrapper.html()).toContain('border')
})

it('renders with proper spacing structure', async () => {
  expect(wrapper.html()).toContain('space-y-3')
})

it('handles long names with line clamp', async () => {
  expect(wrapper.html()).toContain('line-clamp-2')
})
```

**Rationale:** These test implementation details (Tailwind classes) instead of user-visible behavior. They break during styling refactors and provide minimal value.

---

### Category 2: Truthy/Smoke Tests (DELETE)

**Count:** ~24 tests across 6 components

**Examples to delete:**
```typescript
// ❌ DELETE - from SpellCard.test.ts lines 252-289
it('applies correct level color for cantrips', async () => {
  const wrapper = await mountSuspended(SpellCard, { props: { spell: cantrip } })
  expect(wrapper.html()).toBeTruthy() // Meaningless assertion
})

it('applies correct level color for low level spells', async () => {
  expect(wrapper.html()).toBeTruthy() // Just checks it renders
})
```

**Rationale:** If a test successfully mounts the component, it already proves rendering works. Checking `.toBeTruthy()` adds no value.

---

### Category 3: Redundant Shared Behavior (CONSOLIDATE)

**Count:** ~60 tests → ~10 helper functions (called 6 times each)

**Before (6 files × 10 tests = 60 tests):**
```typescript
// SpellCard.test.ts
it('links to spell detail page with slug', async () => { /* ... */ })
it('applies hover effects', async () => { /* ... */ })
it('handles missing sources', async () => { /* ... */ })

// ItemCard.test.ts
it('links to item detail page with slug', async () => { /* ... */ })
it('applies hover effects', async () => { /* ... */ })
it('handles missing sources', async () => { /* ... */ })

// ... same tests in 4 more files
```

**After (1 helper × 6 usages):**
```typescript
// tests/helpers/cardBehavior.ts
export function testCardLinkBehavior(...) { /* ... */ }

// SpellCard.test.ts
testCardLinkBehavior(mountSpellCard, '/spells/fireball')

// ItemCard.test.ts
testCardLinkBehavior(mountItemCard, '/items/longsword')
```

---

## Implementation Plan

### Phase 1: Create Helper Modules

**File 1: `tests/helpers/cardBehavior.ts`** (~80 lines)

Functions to export:
- `testCardLinkBehavior(mountFn, expectedHref)` - Tests slug routing
- `testCardHoverEffects(mountFn)` - Tests hover styling
- `testCardBorderStyling(mountFn)` - Tests card has border (consolidated)

**File 2: `tests/helpers/descriptionBehavior.ts`** (~60 lines)

Functions to export:
- `testDescriptionTruncation(mountLongFn, mountShortFn)` - Tests ellipsis logic
- `testMissingDescriptionFallback(mountFn, expectedDefault)` - Tests default text

**File 3: `tests/helpers/sourceBehavior.ts`** (~40 lines)

Functions to export:
- `testSourceFooter(mountFn, expectedSourceName)` - Tests source display
- `testOptionalSourceFooter(mountFn, entityName)` - Tests missing sources

**Total new code:** ~180 lines of reusable test helpers

---

### Phase 2: Refactor Entity Card Tests

For each of the 6 entity card test files:

1. Import helper functions at top
2. Create `mountEntity()` factory function
3. Replace redundant tests with helper calls (1 line each)
4. Delete implementation detail tests
5. Delete truthy/smoke tests
6. Keep all domain-specific tests

**Per-file changes:**

| File | Current Tests | After Refactor | Tests Removed |
|------|--------------|----------------|---------------|
| SpellCard.test.ts | 29 | 18 | 11 |
| ItemCard.test.ts | 35 | 22 | 13 |
| RaceCard.test.ts | 33 | 20 | 13 |
| ClassCard.test.ts | 30 | 19 | 11 |
| BackgroundCard.test.ts | 26 | 17 | 9 |
| FeatCard.test.ts | 27 | 18 | 9 |
| **TOTAL** | **180** | **114** | **66** |

---

### Phase 3: Verification

After each entity card refactor:

1. Run tests: `docker compose exec nuxt npm test -- <filename>`
2. Verify all remaining tests pass
3. Verify coverage hasn't dropped (should stay ~same)
4. Commit the refactored file

**Safety checks:**
- All tests must pass after each file refactor
- No behavioral tests should be lost (only redundant/trivial ones)
- Test execution time should decrease incrementally

---

## Expected Outcomes

### Quantitative Improvements

- **Tests:** 573 → 507 tests (-11%)
- **Entity card tests:** 180 → 114 tests (-37%)
- **Maintenance files for shared behavior:** 6 → 1 (-83%)
- **Test execution time:** ~3.5s → ~3.0s (-15%)
- **Lines of test code:** ~7,000 → ~6,850 (-2%, but better organized)

### Qualitative Improvements

1. **Reduced maintenance burden** (primary goal)
   - Changing link behavior: Update 1 helper instead of 6 files
   - Changing source footer: Update 1 helper instead of 6 files
   - Refactoring card styling: Update 1 helper instead of 6 files

2. **Better test readability**
   - Entity test files focus on domain logic
   - Shared behavior is explicit (helper function names)
   - Less noise from implementation detail tests

3. **Easier to extend**
   - Adding new entity types: Import helpers, add domain tests
   - New shared patterns: Add to helpers, use across all entities
   - Clear separation of concerns

---

## Risks & Mitigations

### Risk 1: Breaking tests during refactor

**Mitigation:** Refactor one entity at a time, verify tests pass before moving to next

### Risk 2: Losing important test coverage

**Mitigation:** Only delete tests in 3 approved categories (implementation details, truthy assertions, exact duplicates). Keep all domain logic tests.

### Risk 3: Helper functions become too complex

**Mitigation:** Keep helpers simple (1 test per function). If edge cases diverge, keep entity-specific tests instead of parameterizing helpers.

### Risk 4: Team doesn't understand helper pattern

**Mitigation:** Clear documentation in helper files, consistent naming, inline comments explaining usage

---

## Success Criteria

✅ **Test suite passes** - All 507 remaining tests pass
✅ **No coverage loss** - Behavioral coverage remains equivalent
✅ **Maintenance improved** - Shared behavior changes require 1 file update
✅ **Execution faster** - Test run time decreases by 10-20%
✅ **TDD compliance** - Refactored tests still document behavior clearly

---

## Future Opportunities

This design focuses on entity card redundancy, but the pattern could extend to:

1. **Accordion components** (8 files with similar patterns)
2. **List UI components** (EmptyState, ErrorState, Pagination share patterns)
3. **Detail page components** (UiDetailPageHeader, UiDetailPageError)

Estimated additional reduction: ~30-40 more tests if pattern is applied project-wide.

---

## Appendix: Helper Function Signatures

### cardBehavior.ts
```typescript
export function testCardLinkBehavior(
  mountComponent: () => Promise<VueWrapper>,
  expectedHref: string
): void

export function testCardHoverEffects(
  mountComponent: () => Promise<VueWrapper>
): void

export function testCardBorderStyling(
  mountComponent: () => Promise<VueWrapper>
): void
```

### descriptionBehavior.ts
```typescript
export function testDescriptionTruncation(
  mountLongDescription: () => Promise<VueWrapper>,
  mountShortDescription: () => Promise<VueWrapper>
): void

export function testMissingDescriptionFallback(
  mountComponent: () => Promise<VueWrapper>,
  expectedDefault: string
): void
```

### sourceBehavior.ts
```typescript
export function testSourceFooter(
  mountComponent: () => Promise<VueWrapper>,
  expectedSourceName: string
): void

export function testOptionalSourceFooter(
  mountWithoutSources: () => Promise<VueWrapper>,
  entityName: string
): void
```

---

**End of Design Document**
