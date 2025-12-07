# Test Helper Library Design

**Date:** 2025-11-26
**Status:** ✅ Completed
**Effort:** ~8 hours across 4 phases
**Completed:** 2025-11-26

---

## Problem

Store tests have ~2,500 lines of repetitive boilerplate:
- `hasActiveFilters` getter tests: 14 tests × 7 stores = ~100 nearly identical tests
- `activeFilterCount` tests: Similar pattern across stores
- `clearAll` action tests: Same structure everywhere
- `setFromUrlQuery`/`toUrlQuery` tests: Repetitive query object handling
- Mock entity data: 14 card tests with 15-50 lines of mock data each

The existing `tests/helpers/` directory shows excellent patterns for card tests that should be extended to store tests.

---

## Solution Overview

Extend the helper pattern with 4 new helper modules:

| Helper File | Pattern | Lines Saved | Priority |
|-------------|---------|-------------|----------|
| `tests/helpers/storeSetup.ts` | Pinia beforeEach + store assertion utilities | ~300 | Phase 1 |
| `tests/helpers/mockFactories.ts` | Entity mock data factories | ~400 | Phase 1 |
| `tests/helpers/filterStoreBehavior.ts` | Store getter/action test generators | ~800 | Phase 2 |
| `tests/helpers/filterChipBehavior.ts` | Page filter chip test generators | ~300 | Phase 3 |

**Total estimated savings:** ~1,800 lines (current helpers add ~300 lines)
**Net reduction:** ~1,500 lines

---

## Phase 1: Store Setup & Mock Factories (~3 hours)

### 1.1 Store Setup Helper

**Current pattern (repeated 50+ times):**
```typescript
import { setActivePinia, createPinia } from 'pinia'

describe('useSpellFiltersStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })
  // ...
})
```

**Proposed helper:**
```typescript
// tests/helpers/storeSetup.ts
import { setActivePinia, createPinia } from 'pinia'
import { beforeEach } from 'vitest'

/**
 * Sets up fresh Pinia instance before each test.
 * Call at top of describe() block.
 */
export function usePiniaSetup() {
  beforeEach(() => {
    setActivePinia(createPinia())
  })
}

/**
 * Creates and returns a store instance with fresh Pinia.
 * Useful for one-off tests.
 */
export function createTestStore<T>(useStore: () => T): T {
  setActivePinia(createPinia())
  return useStore()
}
```

**After:**
```typescript
import { usePiniaSetup } from '../../helpers/storeSetup'
import { useSpellFiltersStore } from '~/stores/spellFilters'

describe('useSpellFiltersStore', () => {
  usePiniaSetup()
  // ...
})
```

### 1.2 Mock Entity Factories

**Current pattern (repeated per entity):**
```typescript
const mockSpell: Spell = {
  id: 1,
  name: 'Fireball',
  slug: 'fireball',
  level: 3,
  school: { id: 5, code: 'EV', name: 'Evocation' },
  casting_time: '1 action',
  range: '150 feet',
  description: 'A bright streak flashes...',
  is_ritual: false,
  needs_concentration: false,
  sources: [{ code: 'PHB', name: 'Player\'s Handbook', pages: '241' }]
}
```

**Proposed factories:**
```typescript
// tests/helpers/mockFactories.ts
import type { Spell, Item, Monster, CharacterClass, Race, Background, Feat, Source } from '~/types'

// Shared source fixture
export const mockSource: Source = {
  code: 'PHB',
  name: 'Player\'s Handbook',
  pages: '241'
}

export function createMockSpell(overrides: Partial<Spell> = {}): Spell {
  return {
    id: 1,
    name: 'Fireball',
    slug: 'fireball',
    level: 3,
    school: { id: 5, code: 'EV', name: 'Evocation' },
    casting_time: '1 action',
    range: '150 feet',
    description: 'A bright streak flashes from your pointing finger.',
    is_ritual: false,
    needs_concentration: false,
    sources: [mockSource],
    ...overrides
  }
}

export function createMockItem(overrides: Partial<Item> = {}): Item {
  return {
    id: 1,
    name: 'Longsword',
    slug: 'longsword',
    rarity: 'common',
    description: 'A versatile martial weapon.',
    sources: [mockSource],
    ...overrides
  }
}

// Similar factories for: Monster, CharacterClass, Race, Background, Feat
```

**After:**
```typescript
import { createMockSpell } from '../../helpers/mockFactories'

describe('SpellCard', () => {
  const mockSpell = createMockSpell()
  const cantrip = createMockSpell({ level: 0 })
  const ritual = createMockSpell({ is_ritual: true })
  // ...
})
```

---

## Phase 2: Filter Store Behavior (~3 hours)

### 2.1 hasActiveFilters Test Generator

**Current pattern (14 tests per store × 7 stores = ~100 tests):**
```typescript
describe('hasActiveFilters getter', () => {
  it('returns false when no filters active', () => {
    const store = useSpellFiltersStore()
    expect(store.hasActiveFilters).toBe(false)
  })

  it('returns true when searchQuery has value', () => {
    const store = useSpellFiltersStore()
    store.searchQuery = 'fireball'
    expect(store.hasActiveFilters).toBe(true)
  })

  it('returns true when selectedLevels has values', () => {
    const store = useSpellFiltersStore()
    store.selectedLevels = ['3']
    expect(store.hasActiveFilters).toBe(true)
  })
  // ... 10+ more identical patterns
})
```

**Proposed helper:**
```typescript
// tests/helpers/filterStoreBehavior.ts
import { describe, it, expect } from 'vitest'

interface FilterFieldTest {
  field: string
  value: unknown
  label?: string
}

/**
 * Generates hasActiveFilters tests for a filter store.
 * Tests that each field triggers hasActiveFilters when set.
 */
export function testHasActiveFilters<T extends { hasActiveFilters: boolean }>(
  storeName: string,
  createStore: () => T,
  fields: FilterFieldTest[]
) {
  describe('hasActiveFilters getter', () => {
    it('returns false when no filters active', () => {
      const store = createStore()
      expect(store.hasActiveFilters).toBe(false)
    })

    for (const { field, value, label } of fields) {
      it(`returns true when ${label || field} is set`, () => {
        const store = createStore()
        ;(store as Record<string, unknown>)[field] = value
        expect(store.hasActiveFilters).toBe(true)
      })
    }
  })
}
```

**After:**
```typescript
import { testHasActiveFilters } from '../../helpers/filterStoreBehavior'
import { useSpellFiltersStore } from '~/stores/spellFilters'

describe('useSpellFiltersStore', () => {
  usePiniaSetup()

  testHasActiveFilters('SpellFilters', useSpellFiltersStore, [
    { field: 'searchQuery', value: 'fireball' },
    { field: 'selectedLevels', value: ['3'] },
    { field: 'selectedSchool', value: 5 },
    { field: 'selectedClass', value: 'wizard' },
    { field: 'concentrationFilter', value: '1' },
    { field: 'ritualFilter', value: '0' },
    { field: 'selectedDamageTypes', value: ['FIRE'] },
    { field: 'selectedSavingThrows', value: ['DEX'] },
    { field: 'selectedTags', value: ['ritual-caster'] },
    { field: 'verbalFilter', value: '1' },
    { field: 'somaticFilter', value: '1' },
    { field: 'materialFilter', value: '0' },
    { field: 'selectedSources', value: ['PHB'] }
  ])
})
```

**Reduction:** 14 tests → 1 config array (~60 lines → ~15 lines per store)

### 2.2 clearAll Action Test Generator

```typescript
/**
 * Generates clearAll action tests for a filter store.
 */
export function testClearAllAction<T>(
  createStore: () => T,
  setFilters: (store: T) => void,
  expectedDefaults: Partial<T>
) {
  describe('clearAll action', () => {
    it('resets all filters to defaults', () => {
      const store = createStore()
      setFilters(store)
      ;(store as { clearAll: () => void }).clearAll()

      for (const [key, expected] of Object.entries(expectedDefaults)) {
        expect((store as Record<string, unknown>)[key]).toEqual(expected)
      }
    })
  })
}
```

### 2.3 URL Query Sync Test Generator

```typescript
/**
 * Generates setFromUrlQuery and toUrlQuery tests.
 */
export function testUrlQuerySync<T>(
  createStore: () => T,
  testCases: Array<{
    name: string
    query: Record<string, string | string[]>
    expectedState: Partial<T>
  }>
) {
  describe('setFromUrlQuery action', () => {
    for (const { name, query, expectedState } of testCases) {
      it(`handles ${name}`, () => {
        const store = createStore()
        ;(store as { setFromUrlQuery: (q: unknown) => void }).setFromUrlQuery(query)

        for (const [key, expected] of Object.entries(expectedState)) {
          expect((store as Record<string, unknown>)[key]).toEqual(expected)
        }
      })
    }
  })
}
```

---

## Phase 3: Filter Chip Behavior (~1.5 hours)

Similar pattern for page-level filter chip tests:

```typescript
// tests/helpers/filterChipBehavior.ts

/**
 * Tests filter chip display and removal behavior.
 */
export function testFilterChips(
  mountPage: () => Promise<VueWrapper>,
  chips: Array<{
    testId: string
    label: string
    removeAction: (wrapper: VueWrapper) => Promise<void>
    expectedAfterRemove: () => boolean
  }>
) {
  describe('Filter Chips', () => {
    for (const chip of chips) {
      it(`displays ${chip.label} chip`, async () => {
        const wrapper = await mountPage()
        expect(wrapper.find(`[data-testid="${chip.testId}"]`).exists()).toBe(true)
      })

      it(`removes ${chip.label} chip on click`, async () => {
        const wrapper = await mountPage()
        await chip.removeAction(wrapper)
        expect(chip.expectedAfterRemove()).toBe(true)
      })
    }
  })
}
```

---

## Phase 4: Migration & Documentation (~0.5 hours)

1. Update existing tests to use new helpers (incremental, per-store)
2. Add JSDoc documentation to all helpers
3. Update CLAUDE.md test patterns section
4. Create migration guide in docs/

---

## Implementation Tasks

1. **Phase 1A:** Create `tests/helpers/storeSetup.ts`
2. **Phase 1B:** Create `tests/helpers/mockFactories.ts`
3. **Phase 1C:** Migrate 3 entity card tests to use factories (validate pattern)
4. **Phase 2A:** Create `tests/helpers/filterStoreBehavior.ts`
5. **Phase 2B:** Migrate spellFilters.test.ts to new helpers (validate pattern)
6. **Phase 2C:** Migrate remaining 6 store tests
7. **Phase 3:** Create `tests/helpers/filterChipBehavior.ts` + migrate 1 page test
8. **Phase 4:** Documentation and cleanup

---

## Success Metrics

| Metric | Before | After |
|--------|--------|-------|
| Store test lines | ~2,100 | ~800 |
| Mock data duplication | 14 files | 1 factory file |
| New test boilerplate | ~20 lines | ~5 lines |
| Helper files | 3 | 7 |

---

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| Helper complexity hides test intent | Use descriptive function names, keep configs readable |
| Breaking changes during migration | Migrate one file at a time, run tests after each |
| Over-abstraction | Only extract patterns used 3+ times |

---

## Decision: Proceed?

This design extracts clear patterns that are repeated dozens of times. The existing `tests/helpers/` directory proves the pattern works well.

Ready for approval to begin Phase 1.
