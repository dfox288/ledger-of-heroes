# Pinia Filter Store Factory Implementation Plan

**Status:** ✅ Completed
**Completed:** 2025-11-26

**Goal:** Replace 7 nearly-identical Pinia filter stores (1,396 lines) with a single factory function and declarative configurations (~550 lines total, 60% reduction).

**Architecture:** Create a `createEntityFilterStore()` factory that generates stores from field definitions. Each field definition specifies: state name, URL query key, value type (array/single/number/boolean), default value, and whether to persist. The factory handles all getter logic (hasActiveFilters, activeFilterCount, toUrlQuery) and action logic (clearAll, setFromUrlQuery) generically.

**Tech Stack:** TypeScript, Pinia, Vue Router LocationQuery, pinia-plugin-persistedstate, idb-keyval

---

## Current State Analysis

| Store | Lines | Entity-Specific Fields |
|-------|-------|------------------------|
| spellFilters.ts | 229 | 12 filters (levels, school, class, concentration, ritual, damageTypes, savingThrows, tags, verbal, somatic, material) |
| itemFilters.ts | 281 | 15 filters (type, rarity, magic, charges, attunement, stealth, properties, damageTypes, damageDice, versatileDamage, rechargeTiming, strengthReq, range, costRange, acRange) |
| monsterFilters.ts | 264 | 14 filters (CRs, type, legendary, sizes, alignments, movementTypes, armorTypes, hover, lairActions, reactions, spellcaster, magicResistance, acRange, hpRange) |
| classFilters.ts | 158 | 5 filters (baseClass, spellcaster, hitDice, spellcastingAbility, parentClass) |
| raceFilters.ts | 168 | 6 filters (size, speedRange, parentRace, raceType, innateSpells, abilityBonuses) |
| backgroundFilters.ts | 137 | 3 filters (skills, toolTypes, languageChoice) |
| featFilters.ts | 148 | 4 filters (prerequisites, proficiencies, improvedAbilities, prerequisiteTypes) |
| **Total** | **1,385** | **59 unique filters** |

**Shared patterns (repeated 7x):**
- State interface with searchQuery, sortBy, sortDirection, selectedSources, filtersOpen
- DEFAULT_STATE constant
- hasActiveFilters getter (null/empty checks)
- activeFilterCount getter (manual counting)
- toUrlQuery getter (field-to-URL mapping)
- setFromUrlQuery action (URL-to-field parsing)
- clearAll action (identical in all)
- persist config with paths array

---

## Task 1: Create Filter Field Type Definitions

**Files:**
- Create: `app/stores/filterFactory/types.ts`
- Test: `tests/stores/filterFactory/types.test.ts`

**Step 1: Write the type definition test**

```typescript
// tests/stores/filterFactory/types.test.ts
import { describe, it, expect } from 'vitest'
import type { FilterFieldDefinition, FilterFieldType } from '~/stores/filterFactory/types'

describe('FilterFieldDefinition types', () => {
  it('enforces correct field type discriminants', () => {
    // This test validates that TypeScript types are correctly defined
    // by creating valid field definitions
    const stringArrayField: FilterFieldDefinition = {
      name: 'selectedLevels',
      urlKey: 'level',
      type: 'stringArray',
      defaultValue: []
    }

    const numberArrayField: FilterFieldDefinition = {
      name: 'selectedHitDice',
      urlKey: 'hit_die',
      type: 'numberArray',
      defaultValue: []
    }

    const singleStringField: FilterFieldDefinition = {
      name: 'concentrationFilter',
      urlKey: 'concentration',
      type: 'string',
      defaultValue: null
    }

    const singleNumberField: FilterFieldDefinition = {
      name: 'selectedSchool',
      urlKey: 'school',
      type: 'number',
      defaultValue: null
    }

    const emptyStringField: FilterFieldDefinition = {
      name: 'selectedSize',
      urlKey: 'size',
      type: 'emptyString',
      defaultValue: ''
    }

    expect(stringArrayField.type).toBe('stringArray')
    expect(numberArrayField.type).toBe('numberArray')
    expect(singleStringField.type).toBe('string')
    expect(singleNumberField.type).toBe('number')
    expect(emptyStringField.type).toBe('emptyString')
  })

  it('allows optional persist flag (defaults to true)', () => {
    const fieldWithPersist: FilterFieldDefinition = {
      name: 'test',
      urlKey: 'test',
      type: 'string',
      defaultValue: null,
      persist: false
    }

    const fieldWithoutPersist: FilterFieldDefinition = {
      name: 'test',
      urlKey: 'test',
      type: 'string',
      defaultValue: null
    }

    expect(fieldWithPersist.persist).toBe(false)
    expect(fieldWithoutPersist.persist).toBeUndefined()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/filterFactory/types.test.ts`
Expected: FAIL - Cannot find module '~/stores/filterFactory/types'

**Step 3: Write the type definitions**

```typescript
// app/stores/filterFactory/types.ts

/**
 * Supported filter field types:
 * - 'stringArray': string[] - multiple string selections (e.g., selectedLevels, selectedSources)
 * - 'numberArray': number[] - multiple number selections (e.g., selectedHitDice)
 * - 'string': string | null - single string value (e.g., concentrationFilter: '1' | '0' | null)
 * - 'number': number | null - single number value (e.g., selectedSchool: 5 | null)
 * - 'emptyString': string - single string with '' as empty (e.g., selectedSize: 'M' | '')
 */
export type FilterFieldType = 'stringArray' | 'numberArray' | 'string' | 'number' | 'emptyString'

/**
 * Definition for a single filter field.
 * Used by the factory to generate state, getters, and actions.
 */
export interface FilterFieldDefinition {
  /** State property name (e.g., 'selectedLevels') */
  name: string
  /** URL query parameter key (e.g., 'level') */
  urlKey: string
  /** Field value type - determines parsing and empty-checking logic */
  type: FilterFieldType
  /** Default value when store is reset */
  defaultValue: string[] | number[] | string | number | null
  /** Whether to persist in IndexedDB (default: true) */
  persist?: boolean
}

/**
 * Configuration for creating an entity filter store.
 */
export interface EntityFilterStoreConfig {
  /** Pinia store name (e.g., 'spellFilters') */
  name: string
  /** Storage key for IndexedDB persistence */
  storageKey: string
  /** Entity-specific filter field definitions */
  fields: FilterFieldDefinition[]
}

/**
 * Base state shared by all entity filter stores.
 * These fields are NOT configured per-entity.
 */
export interface BaseFilterState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  filtersOpen: boolean
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/filterFactory/types.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/filterFactory/types.ts tests/stores/filterFactory/types.test.ts
git commit -m "$(cat <<'EOF'
feat(stores): Add filter factory type definitions

- FilterFieldType: stringArray, numberArray, string, number, emptyString
- FilterFieldDefinition: name, urlKey, type, defaultValue, persist
- EntityFilterStoreConfig: name, storageKey, fields
- BaseFilterState: searchQuery, sortBy, sortDirection, selectedSources, filtersOpen

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Create Filter Utility Functions

**Files:**
- Create: `app/stores/filterFactory/utils.ts`
- Test: `tests/stores/filterFactory/utils.test.ts`

**Step 1: Write the utility tests**

```typescript
// tests/stores/filterFactory/utils.test.ts
import { describe, it, expect } from 'vitest'
import {
  isFieldEmpty,
  countFieldValue,
  fieldToUrlValue,
  urlValueToField
} from '~/stores/filterFactory/utils'
import type { FilterFieldDefinition } from '~/stores/filterFactory/types'

describe('isFieldEmpty', () => {
  describe('stringArray type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'stringArray', defaultValue: []
    }

    it('returns true for empty array', () => {
      expect(isFieldEmpty(field, [])).toBe(true)
    })

    it('returns false for non-empty array', () => {
      expect(isFieldEmpty(field, ['a', 'b'])).toBe(false)
    })
  })

  describe('numberArray type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'numberArray', defaultValue: []
    }

    it('returns true for empty array', () => {
      expect(isFieldEmpty(field, [])).toBe(true)
    })

    it('returns false for non-empty array', () => {
      expect(isFieldEmpty(field, [1, 2])).toBe(false)
    })
  })

  describe('string type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'string', defaultValue: null
    }

    it('returns true for null', () => {
      expect(isFieldEmpty(field, null)).toBe(true)
    })

    it('returns false for string value', () => {
      expect(isFieldEmpty(field, '1')).toBe(false)
    })
  })

  describe('number type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'number', defaultValue: null
    }

    it('returns true for null', () => {
      expect(isFieldEmpty(field, null)).toBe(true)
    })

    it('returns false for number value', () => {
      expect(isFieldEmpty(field, 5)).toBe(false)
    })

    it('returns false for zero', () => {
      expect(isFieldEmpty(field, 0)).toBe(false)
    })
  })

  describe('emptyString type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'emptyString', defaultValue: ''
    }

    it('returns true for empty string', () => {
      expect(isFieldEmpty(field, '')).toBe(true)
    })

    it('returns false for non-empty string', () => {
      expect(isFieldEmpty(field, 'M')).toBe(false)
    })
  })
})

describe('countFieldValue', () => {
  it('counts array length for stringArray', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'stringArray', defaultValue: []
    }
    expect(countFieldValue(field, ['a', 'b', 'c'])).toBe(3)
    expect(countFieldValue(field, [])).toBe(0)
  })

  it('counts array length for numberArray', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'numberArray', defaultValue: []
    }
    expect(countFieldValue(field, [1, 2])).toBe(2)
  })

  it('counts 1 for non-null single values', () => {
    const stringField: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'string', defaultValue: null
    }
    expect(countFieldValue(stringField, '1')).toBe(1)
    expect(countFieldValue(stringField, null)).toBe(0)
  })

  it('counts 1 for non-empty emptyString type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'emptyString', defaultValue: ''
    }
    expect(countFieldValue(field, 'M')).toBe(1)
    expect(countFieldValue(field, '')).toBe(0)
  })
})

describe('fieldToUrlValue', () => {
  it('returns string array for stringArray type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'stringArray', defaultValue: []
    }
    expect(fieldToUrlValue(field, ['a', 'b'])).toEqual(['a', 'b'])
  })

  it('converts numberArray to string array', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'numberArray', defaultValue: []
    }
    expect(fieldToUrlValue(field, [6, 8, 10])).toEqual(['6', '8', '10'])
  })

  it('converts number to string for number type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'number', defaultValue: null
    }
    expect(fieldToUrlValue(field, 5)).toBe('5')
  })

  it('returns string as-is for string type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'string', defaultValue: null
    }
    expect(fieldToUrlValue(field, '1')).toBe('1')
  })

  it('returns string as-is for emptyString type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'emptyString', defaultValue: ''
    }
    expect(fieldToUrlValue(field, 'M')).toBe('M')
  })
})

describe('urlValueToField', () => {
  describe('stringArray type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'stringArray', defaultValue: []
    }

    it('converts single string to array', () => {
      expect(urlValueToField(field, 'a')).toEqual(['a'])
    })

    it('keeps array as string array', () => {
      expect(urlValueToField(field, ['a', 'b'])).toEqual(['a', 'b'])
    })
  })

  describe('numberArray type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'numberArray', defaultValue: []
    }

    it('converts single string to number array', () => {
      expect(urlValueToField(field, '6')).toEqual([6])
    })

    it('converts string array to number array', () => {
      expect(urlValueToField(field, ['6', '8'])).toEqual([6, 8])
    })
  })

  describe('number type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'number', defaultValue: null
    }

    it('converts string to number', () => {
      expect(urlValueToField(field, '5')).toBe(5)
    })

    it('takes first element if array', () => {
      expect(urlValueToField(field, ['5', '6'])).toBe(5)
    })
  })

  describe('string type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'string', defaultValue: null
    }

    it('returns string as-is', () => {
      expect(urlValueToField(field, '1')).toBe('1')
    })

    it('takes first element if array', () => {
      expect(urlValueToField(field, ['1', '0'])).toBe('1')
    })
  })

  describe('emptyString type', () => {
    const field: FilterFieldDefinition = {
      name: 'test', urlKey: 'test', type: 'emptyString', defaultValue: ''
    }

    it('returns string as-is', () => {
      expect(urlValueToField(field, 'M')).toBe('M')
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/filterFactory/utils.test.ts`
Expected: FAIL - Cannot find module '~/stores/filterFactory/utils'

**Step 3: Write the utility functions**

```typescript
// app/stores/filterFactory/utils.ts
import type { FilterFieldDefinition } from './types'

/**
 * Check if a field value is "empty" (should not trigger hasActiveFilters).
 */
export function isFieldEmpty(field: FilterFieldDefinition, value: unknown): boolean {
  switch (field.type) {
    case 'stringArray':
    case 'numberArray':
      return !Array.isArray(value) || value.length === 0
    case 'string':
    case 'number':
      return value === null
    case 'emptyString':
      return value === ''
    default:
      return true
  }
}

/**
 * Count the "active" filter count for a field value.
 * Arrays count their length, single values count as 1 if set.
 */
export function countFieldValue(field: FilterFieldDefinition, value: unknown): number {
  switch (field.type) {
    case 'stringArray':
    case 'numberArray':
      return Array.isArray(value) ? value.length : 0
    case 'string':
    case 'number':
      return value !== null ? 1 : 0
    case 'emptyString':
      return value !== '' ? 1 : 0
    default:
      return 0
  }
}

/**
 * Convert a field value to URL query format.
 * Numbers are stringified, arrays are kept as arrays.
 */
export function fieldToUrlValue(
  field: FilterFieldDefinition,
  value: unknown
): string | string[] {
  switch (field.type) {
    case 'stringArray':
      return value as string[]
    case 'numberArray':
      return (value as number[]).map(String)
    case 'number':
      return String(value)
    case 'string':
    case 'emptyString':
      return value as string
    default:
      return String(value)
  }
}

/**
 * Convert a URL query value to the field's expected type.
 * Handles both string and string[] from vue-router LocationQuery.
 */
export function urlValueToField(
  field: FilterFieldDefinition,
  urlValue: string | string[]
): string[] | number[] | string | number {
  const firstValue = Array.isArray(urlValue) ? urlValue[0] : urlValue

  switch (field.type) {
    case 'stringArray':
      return Array.isArray(urlValue) ? urlValue.map(String) : [String(urlValue)]
    case 'numberArray':
      return Array.isArray(urlValue) ? urlValue.map(Number) : [Number(urlValue)]
    case 'number':
      return Number(firstValue)
    case 'string':
    case 'emptyString':
      return String(firstValue)
    default:
      return String(firstValue)
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/filterFactory/utils.test.ts`
Expected: PASS (all 24 tests)

**Step 5: Commit**

```bash
git add app/stores/filterFactory/utils.ts tests/stores/filterFactory/utils.test.ts
git commit -m "$(cat <<'EOF'
feat(stores): Add filter factory utility functions

- isFieldEmpty: Check if field value triggers hasActiveFilters
- countFieldValue: Count active filter count for a field
- fieldToUrlValue: Convert state value to URL query format
- urlValueToField: Parse URL query value to state format

Handles all 5 field types: stringArray, numberArray, string, number, emptyString

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Create the Store Factory Function

**Files:**
- Create: `app/stores/filterFactory/createEntityFilterStore.ts`
- Test: `tests/stores/filterFactory/createEntityFilterStore.test.ts`

**Step 1: Write the factory tests**

```typescript
// tests/stores/filterFactory/createEntityFilterStore.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { createEntityFilterStore } from '~/stores/filterFactory/createEntityFilterStore'
import type { EntityFilterStoreConfig } from '~/stores/filterFactory/types'

// Simple test store config
const testConfig: EntityFilterStoreConfig = {
  name: 'testFilters',
  storageKey: 'test-filters',
  fields: [
    { name: 'selectedLevels', urlKey: 'level', type: 'stringArray', defaultValue: [] },
    { name: 'selectedSchool', urlKey: 'school', type: 'number', defaultValue: null },
    { name: 'concentrationFilter', urlKey: 'concentration', type: 'string', defaultValue: null },
    { name: 'selectedSize', urlKey: 'size', type: 'emptyString', defaultValue: '' },
    { name: 'selectedHitDice', urlKey: 'hit_die', type: 'numberArray', defaultValue: [] }
  ]
}

describe('createEntityFilterStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  describe('state initialization', () => {
    it('creates store with base state', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()

      expect(store.searchQuery).toBe('')
      expect(store.sortBy).toBe('name')
      expect(store.sortDirection).toBe('asc')
      expect(store.selectedSources).toEqual([])
      expect(store.filtersOpen).toBe(false)
    })

    it('creates store with entity-specific fields from config', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()

      expect(store.selectedLevels).toEqual([])
      expect(store.selectedSchool).toBeNull()
      expect(store.concentrationFilter).toBeNull()
      expect(store.selectedSize).toBe('')
      expect(store.selectedHitDice).toEqual([])
    })
  })

  describe('hasActiveFilters getter', () => {
    it('returns false when no filters active', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()

      expect(store.hasActiveFilters).toBe(false)
    })

    it('returns true when searchQuery has value', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.searchQuery = 'test'

      expect(store.hasActiveFilters).toBe(true)
    })

    it('returns true when selectedSources has values', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.selectedSources = ['PHB']

      expect(store.hasActiveFilters).toBe(true)
    })

    it('returns true when stringArray field has values', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.selectedLevels = ['3']

      expect(store.hasActiveFilters).toBe(true)
    })

    it('returns true when number field is set', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.selectedSchool = 5

      expect(store.hasActiveFilters).toBe(true)
    })

    it('returns true when string field is set', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.concentrationFilter = '1'

      expect(store.hasActiveFilters).toBe(true)
    })

    it('returns true when emptyString field has value', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.selectedSize = 'M'

      expect(store.hasActiveFilters).toBe(true)
    })

    it('returns true when numberArray field has values', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.selectedHitDice = [6, 8]

      expect(store.hasActiveFilters).toBe(true)
    })
  })

  describe('activeFilterCount getter', () => {
    it('returns 0 when no filters active', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()

      expect(store.activeFilterCount).toBe(0)
    })

    it('does not count searchQuery', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.searchQuery = 'test'

      expect(store.activeFilterCount).toBe(0)
    })

    it('counts array lengths and single values', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.selectedSources = ['PHB', 'XGE']  // 2
      store.selectedLevels = ['1', '2', '3']   // 3
      store.selectedSchool = 5                  // 1
      store.concentrationFilter = '1'           // 1
      store.selectedSize = 'M'                  // 1
      store.selectedHitDice = [6, 8]           // 2

      expect(store.activeFilterCount).toBe(10)
    })
  })

  describe('toUrlQuery getter', () => {
    it('returns empty object when no filters active', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()

      expect(store.toUrlQuery).toEqual({})
    })

    it('includes active entity filters in URL query', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.selectedLevels = ['3', '5']
      store.selectedSchool = 5
      store.concentrationFilter = '1'
      store.selectedSize = 'M'
      store.selectedHitDice = [6, 8]

      const query = store.toUrlQuery
      expect(query.level).toEqual(['3', '5'])
      expect(query.school).toBe('5')
      expect(query.concentration).toBe('1')
      expect(query.size).toBe('M')
      expect(query.hit_die).toEqual(['6', '8'])
    })

    it('includes sources in URL query', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.selectedSources = ['PHB']

      expect(store.toUrlQuery.source).toEqual(['PHB'])
    })

    it('includes non-default sort values', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.sortBy = 'level'
      store.sortDirection = 'desc'

      expect(store.toUrlQuery.sort_by).toBe('level')
      expect(store.toUrlQuery.sort_direction).toBe('desc')
    })

    it('excludes default sort values', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()

      expect(store.toUrlQuery.sort_by).toBeUndefined()
      expect(store.toUrlQuery.sort_direction).toBeUndefined()
    })
  })

  describe('clearAll action', () => {
    it('resets all filters to defaults', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()

      store.searchQuery = 'test'
      store.sortBy = 'level'
      store.selectedSources = ['PHB']
      store.selectedLevels = ['3']
      store.selectedSchool = 5
      store.filtersOpen = true

      store.clearAll()

      expect(store.searchQuery).toBe('')
      expect(store.sortBy).toBe('name')
      expect(store.selectedSources).toEqual([])
      expect(store.selectedLevels).toEqual([])
      expect(store.selectedSchool).toBeNull()
    })

    it('preserves filtersOpen state', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()
      store.filtersOpen = true
      store.searchQuery = 'test'

      store.clearAll()

      expect(store.filtersOpen).toBe(true)
    })
  })

  describe('setFromUrlQuery action', () => {
    it('sets filters from URL query', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()

      store.setFromUrlQuery({
        level: ['3', '5'],
        school: '5',
        concentration: '1',
        size: 'M',
        hit_die: ['6', '8'],
        source: 'PHB',
        sort_by: 'level',
        sort_direction: 'desc'
      })

      expect(store.selectedLevels).toEqual(['3', '5'])
      expect(store.selectedSchool).toBe(5)
      expect(store.concentrationFilter).toBe('1')
      expect(store.selectedSize).toBe('M')
      expect(store.selectedHitDice).toEqual([6, 8])
      expect(store.selectedSources).toEqual(['PHB'])
      expect(store.sortBy).toBe('level')
      expect(store.sortDirection).toBe('desc')
    })

    it('handles single values as arrays for array types', () => {
      const useStore = createEntityFilterStore(testConfig)
      const store = useStore()

      store.setFromUrlQuery({ level: '3' })
      expect(store.selectedLevels).toEqual(['3'])

      store.setFromUrlQuery({ source: 'PHB' })
      expect(store.selectedSources).toEqual(['PHB'])
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/filterFactory/createEntityFilterStore.test.ts`
Expected: FAIL - Cannot find module '~/stores/filterFactory/createEntityFilterStore'

**Step 3: Write the factory function**

```typescript
// app/stores/filterFactory/createEntityFilterStore.ts
import { defineStore } from 'pinia'
import type { LocationQuery } from 'vue-router'
import { idbStorage } from '~/utils/idbStorage'
import type { EntityFilterStoreConfig, BaseFilterState } from './types'
import { isFieldEmpty, countFieldValue, fieldToUrlValue, urlValueToField } from './utils'

/**
 * Creates a Pinia store for entity filtering with URL sync and persistence.
 *
 * @param config - Store configuration with name, storage key, and field definitions
 * @returns A Pinia store definition function (useXxxFiltersStore)
 *
 * @example
 * ```ts
 * export const useSpellFiltersStore = createEntityFilterStore({
 *   name: 'spellFilters',
 *   storageKey: 'dnd-filters-spells',
 *   fields: [
 *     { name: 'selectedLevels', urlKey: 'level', type: 'stringArray', defaultValue: [] },
 *     { name: 'selectedSchool', urlKey: 'school', type: 'number', defaultValue: null },
 *   ]
 * })
 * ```
 */
export function createEntityFilterStore(config: EntityFilterStoreConfig) {
  // Build default state from base + config fields
  const defaultEntityState: Record<string, unknown> = {}
  for (const field of config.fields) {
    defaultEntityState[field.name] = field.defaultValue
  }

  type StoreState = BaseFilterState & Record<string, unknown>

  const DEFAULT_STATE: StoreState = {
    // Base state (same for all entities)
    searchQuery: '',
    sortBy: 'name',
    sortDirection: 'asc' as const,
    selectedSources: [] as string[],
    filtersOpen: false,
    // Entity-specific state from config
    ...defaultEntityState
  }

  return defineStore(config.name, {
    state: (): StoreState => ({
      ...DEFAULT_STATE,
      // Deep clone arrays to prevent reference sharing
      selectedSources: [],
      ...Object.fromEntries(
        config.fields.map(f =>
          [f.name, Array.isArray(f.defaultValue) ? [...f.defaultValue] : f.defaultValue]
        )
      )
    }),

    getters: {
      hasActiveFilters: (state): boolean => {
        // Check base fields
        if (state.searchQuery !== '') return true
        if (state.selectedSources.length > 0) return true

        // Check entity-specific fields
        for (const field of config.fields) {
          if (!isFieldEmpty(field, state[field.name])) {
            return true
          }
        }

        return false
      },

      activeFilterCount: (state): number => {
        let count = 0

        // Count sources (but not searchQuery)
        count += state.selectedSources.length

        // Count entity-specific fields
        for (const field of config.fields) {
          count += countFieldValue(field, state[field.name])
        }

        return count
      },

      toUrlQuery: (state): Record<string, string | string[]> => {
        const query: Record<string, string | string[]> = {}

        // Entity-specific fields
        for (const field of config.fields) {
          if (!isFieldEmpty(field, state[field.name])) {
            query[field.urlKey] = fieldToUrlValue(field, state[field.name])
          }
        }

        // Sources
        if (state.selectedSources.length > 0) {
          query.source = state.selectedSources
        }

        // Sort (only if non-default)
        if (state.sortBy !== 'name') {
          query.sort_by = state.sortBy
        }
        if (state.sortDirection !== 'asc') {
          query.sort_direction = state.sortDirection
        }

        return query
      }
    },

    actions: {
      clearAll() {
        // Preserve filtersOpen (UI preference)
        const filtersOpen = this.filtersOpen
        this.$reset()
        this.filtersOpen = filtersOpen
      },

      setFromUrlQuery(query: LocationQuery) {
        // Parse entity-specific fields
        for (const field of config.fields) {
          const urlValue = query[field.urlKey]
          if (urlValue !== undefined && urlValue !== null) {
            (this as Record<string, unknown>)[field.name] = urlValueToField(field, urlValue as string | string[])
          }
        }

        // Parse sources
        if (query.source) {
          this.selectedSources = Array.isArray(query.source)
            ? query.source.map(String)
            : [String(query.source)]
        }

        // Parse sort
        if (query.sort_by) {
          this.sortBy = String(query.sort_by)
        }
        if (query.sort_direction) {
          this.sortDirection = query.sort_direction as 'asc' | 'desc'
        }
      }
    },

    persist: {
      key: config.storageKey,
      storage: idbStorage,
      // Persist all fields except searchQuery
      paths: [
        'sortBy',
        'sortDirection',
        'selectedSources',
        'filtersOpen',
        ...config.fields.filter(f => f.persist !== false).map(f => f.name)
      ]
    }
  })
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/filterFactory/createEntityFilterStore.test.ts`
Expected: PASS (all tests)

**Step 5: Commit**

```bash
git add app/stores/filterFactory/createEntityFilterStore.ts tests/stores/filterFactory/createEntityFilterStore.test.ts
git commit -m "$(cat <<'EOF'
feat(stores): Add createEntityFilterStore factory function

Factory generates complete Pinia stores from declarative config:
- State: base fields + entity-specific from config
- Getters: hasActiveFilters, activeFilterCount, toUrlQuery
- Actions: clearAll, setFromUrlQuery
- Persistence: IndexedDB via pinia-plugin-persistedstate

Eliminates ~150 lines of boilerplate per store.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Create Index File and Export Factory

**Files:**
- Create: `app/stores/filterFactory/index.ts`

**Step 1: Create the index file**

```typescript
// app/stores/filterFactory/index.ts
export { createEntityFilterStore } from './createEntityFilterStore'
export type {
  FilterFieldType,
  FilterFieldDefinition,
  EntityFilterStoreConfig,
  BaseFilterState
} from './types'
```

**Step 2: Commit**

```bash
git add app/stores/filterFactory/index.ts
git commit -m "$(cat <<'EOF'
feat(stores): Add filterFactory index exports

Re-exports createEntityFilterStore and type definitions.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Migrate spellFilters to Factory Pattern

**Files:**
- Modify: `app/stores/spellFilters.ts`
- Modify: `tests/stores/spellFilters.test.ts`

**Step 1: Run existing tests to verify baseline**

Run: `docker compose exec nuxt npm run test -- tests/stores/spellFilters.test.ts`
Expected: PASS (all 21 tests pass - this is our baseline)

**Step 2: Rewrite spellFilters.ts using factory**

```typescript
// app/stores/spellFilters.ts
import { createEntityFilterStore } from './filterFactory'
import { STORE_KEYS } from './types'

/**
 * Spell filter store - manages all filter state for the spells list page.
 *
 * Uses the filter factory for consistent behavior with other entity stores.
 */
export const useSpellFiltersStore = createEntityFilterStore({
  name: 'spellFilters',
  storageKey: STORE_KEYS.spells,
  fields: [
    // Level filter (multi-select: "0", "1", "2", ..., "9")
    { name: 'selectedLevels', urlKey: 'level', type: 'stringArray', defaultValue: [] },
    // School filter (single select by ID)
    { name: 'selectedSchool', urlKey: 'school', type: 'number', defaultValue: null },
    // Class filter (single select by slug)
    { name: 'selectedClass', urlKey: 'class', type: 'string', defaultValue: null },
    // Concentration toggle ("1" = yes, "0" = no, null = any)
    { name: 'concentrationFilter', urlKey: 'concentration', type: 'string', defaultValue: null },
    // Ritual toggle
    { name: 'ritualFilter', urlKey: 'ritual', type: 'string', defaultValue: null },
    // Damage types (multi-select by code: "FIRE", "COLD", etc.)
    { name: 'selectedDamageTypes', urlKey: 'damage_type', type: 'stringArray', defaultValue: [] },
    // Saving throws (multi-select by code: "DEX", "WIS", etc.)
    { name: 'selectedSavingThrows', urlKey: 'saving_throw', type: 'stringArray', defaultValue: [] },
    // Tags (multi-select)
    { name: 'selectedTags', urlKey: 'tag', type: 'stringArray', defaultValue: [] },
    // Component toggles
    { name: 'verbalFilter', urlKey: 'has_verbal', type: 'string', defaultValue: null },
    { name: 'somaticFilter', urlKey: 'has_somatic', type: 'string', defaultValue: null },
    { name: 'materialFilter', urlKey: 'has_material', type: 'string', defaultValue: null }
  ]
})

// Re-export interface for backwards compatibility
// (can be removed once all consumers use store directly)
export interface SpellFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedLevels: string[]
  selectedSchool: number | null
  selectedClass: string | null
  concentrationFilter: string | null
  ritualFilter: string | null
  selectedDamageTypes: string[]
  selectedSavingThrows: string[]
  selectedTags: string[]
  verbalFilter: string | null
  somaticFilter: string | null
  materialFilter: string | null
  filtersOpen: boolean
}
```

**Step 3: Run tests to verify migration**

Run: `docker compose exec nuxt npm run test -- tests/stores/spellFilters.test.ts`
Expected: PASS (all 21 tests should pass without modification)

**Step 4: Run the spells test suite to check integration**

Run: `docker compose exec nuxt npm run test:spells`
Expected: PASS (all spells domain tests pass)

**Step 5: Commit**

```bash
git add app/stores/spellFilters.ts
git commit -m "$(cat <<'EOF'
refactor(stores): Migrate spellFilters to factory pattern

Replaces 229 lines with ~50 lines using createEntityFilterStore.
All existing tests pass without modification.

Before: Manual state, getters, actions, persist config
After: Declarative field definitions

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Migrate itemFilters to Factory Pattern

**Files:**
- Modify: `app/stores/itemFilters.ts`

**Step 1: Run existing tests to verify baseline**

Run: `docker compose exec nuxt npm run test -- tests/stores/itemFilters.test.ts`
Expected: PASS

**Step 2: Rewrite itemFilters.ts using factory**

```typescript
// app/stores/itemFilters.ts
import { createEntityFilterStore } from './filterFactory'
import { STORE_KEYS } from './types'

export const useItemFiltersStore = createEntityFilterStore({
  name: 'itemFilters',
  storageKey: STORE_KEYS.items,
  fields: [
    // Primary filters
    { name: 'selectedType', urlKey: 'type', type: 'number', defaultValue: null },
    { name: 'selectedRarity', urlKey: 'rarity', type: 'string', defaultValue: null },
    { name: 'selectedMagic', urlKey: 'is_magic', type: 'string', defaultValue: null },
    // Quick toggles
    { name: 'hasCharges', urlKey: 'has_charges', type: 'string', defaultValue: null },
    { name: 'requiresAttunement', urlKey: 'requires_attunement', type: 'string', defaultValue: null },
    { name: 'stealthDisadvantage', urlKey: 'stealth_disadvantage', type: 'string', defaultValue: null },
    // Advanced arrays
    { name: 'selectedProperties', urlKey: 'property', type: 'stringArray', defaultValue: [] },
    { name: 'selectedDamageTypes', urlKey: 'damage_type', type: 'stringArray', defaultValue: [] },
    { name: 'selectedDamageDice', urlKey: 'damage_dice', type: 'stringArray', defaultValue: [] },
    { name: 'selectedVersatileDamage', urlKey: 'versatile_damage', type: 'stringArray', defaultValue: [] },
    { name: 'selectedRechargeTiming', urlKey: 'recharge_timing', type: 'stringArray', defaultValue: [] },
    // Advanced singles
    { name: 'selectedStrengthReq', urlKey: 'strength_req', type: 'string', defaultValue: null },
    { name: 'selectedRange', urlKey: 'range', type: 'string', defaultValue: null },
    { name: 'selectedCostRange', urlKey: 'cost', type: 'string', defaultValue: null },
    { name: 'selectedACRange', urlKey: 'ac', type: 'string', defaultValue: null }
  ]
})

export interface ItemFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedType: number | null
  selectedRarity: string | null
  selectedMagic: string | null
  hasCharges: string | null
  requiresAttunement: string | null
  stealthDisadvantage: string | null
  selectedProperties: string[]
  selectedDamageTypes: string[]
  selectedDamageDice: string[]
  selectedVersatileDamage: string[]
  selectedRechargeTiming: string[]
  selectedStrengthReq: string | null
  selectedRange: string | null
  selectedCostRange: string | null
  selectedACRange: string | null
  filtersOpen: boolean
}
```

**Step 3: Run tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/itemFilters.test.ts`
Expected: PASS

**Step 4: Run items domain tests**

Run: `docker compose exec nuxt npm run test:items`
Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/itemFilters.ts
git commit -m "$(cat <<'EOF'
refactor(stores): Migrate itemFilters to factory pattern

Replaces 281 lines with ~55 lines using createEntityFilterStore.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Migrate monsterFilters to Factory Pattern

**Files:**
- Modify: `app/stores/monsterFilters.ts`

**Step 1: Run existing tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/monsterFilters.test.ts`
Expected: PASS

**Step 2: Rewrite using factory**

```typescript
// app/stores/monsterFilters.ts
import { createEntityFilterStore } from './filterFactory'
import { STORE_KEYS } from './types'

export const useMonsterFiltersStore = createEntityFilterStore({
  name: 'monsterFilters',
  storageKey: STORE_KEYS.monsters,
  fields: [
    { name: 'selectedCRs', urlKey: 'cr', type: 'stringArray', defaultValue: [] },
    { name: 'selectedType', urlKey: 'type', type: 'string', defaultValue: null },
    { name: 'isLegendary', urlKey: 'is_legendary', type: 'string', defaultValue: null },
    { name: 'selectedSizes', urlKey: 'size_id', type: 'stringArray', defaultValue: [] },
    { name: 'selectedAlignments', urlKey: 'alignment', type: 'stringArray', defaultValue: [] },
    { name: 'selectedMovementTypes', urlKey: 'movement', type: 'stringArray', defaultValue: [] },
    { name: 'selectedArmorTypes', urlKey: 'armor_type', type: 'stringArray', defaultValue: [] },
    { name: 'canHover', urlKey: 'can_hover', type: 'string', defaultValue: null },
    { name: 'hasLairActions', urlKey: 'has_lair_actions', type: 'string', defaultValue: null },
    { name: 'hasReactions', urlKey: 'has_reactions', type: 'string', defaultValue: null },
    { name: 'isSpellcaster', urlKey: 'is_spellcaster', type: 'string', defaultValue: null },
    { name: 'hasMagicResistance', urlKey: 'has_magic_resistance', type: 'string', defaultValue: null },
    { name: 'selectedACRange', urlKey: 'ac_range', type: 'string', defaultValue: null },
    { name: 'selectedHPRange', urlKey: 'hp_range', type: 'string', defaultValue: null }
  ]
})

export interface MonsterFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedCRs: string[]
  selectedType: string | null
  isLegendary: string | null
  selectedSizes: string[]
  selectedAlignments: string[]
  selectedMovementTypes: string[]
  selectedArmorTypes: string[]
  canHover: string | null
  hasLairActions: string | null
  hasReactions: string | null
  isSpellcaster: string | null
  hasMagicResistance: string | null
  selectedACRange: string | null
  selectedHPRange: string | null
  filtersOpen: boolean
}
```

**Step 3: Run tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/monsterFilters.test.ts && docker compose exec nuxt npm run test:monsters`
Expected: PASS

**Step 4: Commit**

```bash
git add app/stores/monsterFilters.ts
git commit -m "$(cat <<'EOF'
refactor(stores): Migrate monsterFilters to factory pattern

Replaces 264 lines with ~50 lines.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Migrate classFilters to Factory Pattern

**Files:**
- Modify: `app/stores/classFilters.ts`

**Step 1: Run existing tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/classFilters.test.ts`

**Step 2: Rewrite using factory**

```typescript
// app/stores/classFilters.ts
import { createEntityFilterStore } from './filterFactory'
import { STORE_KEYS } from './types'

export const useClassFiltersStore = createEntityFilterStore({
  name: 'classFilters',
  storageKey: STORE_KEYS.classes,
  fields: [
    { name: 'isBaseClass', urlKey: 'is_base_class', type: 'string', defaultValue: null },
    { name: 'isSpellcaster', urlKey: 'is_spellcaster', type: 'string', defaultValue: null },
    { name: 'selectedHitDice', urlKey: 'hit_die', type: 'numberArray', defaultValue: [] },
    { name: 'selectedSpellcastingAbility', urlKey: 'spellcasting_ability', type: 'string', defaultValue: null },
    { name: 'selectedParentClass', urlKey: 'parent_class_name', type: 'string', defaultValue: null }
  ]
})

export interface ClassFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  isBaseClass: string | null
  isSpellcaster: string | null
  selectedHitDice: number[]
  selectedSpellcastingAbility: string | null
  selectedParentClass: string | null
  filtersOpen: boolean
}
```

**Step 3: Run tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/classFilters.test.ts && docker compose exec nuxt npm run test:classes`

**Step 4: Commit**

```bash
git add app/stores/classFilters.ts
git commit -m "$(cat <<'EOF'
refactor(stores): Migrate classFilters to factory pattern

Replaces 158 lines with ~35 lines.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 9: Migrate raceFilters to Factory Pattern

**Files:**
- Modify: `app/stores/raceFilters.ts`

**Step 1: Run existing tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/raceFilters.test.ts`

**Step 2: Rewrite using factory**

```typescript
// app/stores/raceFilters.ts
import { createEntityFilterStore } from './filterFactory'
import { STORE_KEYS } from './types'

export const useRaceFiltersStore = createEntityFilterStore({
  name: 'raceFilters',
  storageKey: STORE_KEYS.races,
  fields: [
    // Note: selectedSize uses emptyString type ('' = any, 'M' = medium, etc.)
    { name: 'selectedSize', urlKey: 'size', type: 'emptyString', defaultValue: '' },
    { name: 'selectedSpeedRange', urlKey: 'speed', type: 'string', defaultValue: null },
    // Note: selectedParentRace uses emptyString type
    { name: 'selectedParentRace', urlKey: 'parent_race', type: 'emptyString', defaultValue: '' },
    { name: 'raceTypeFilter', urlKey: 'race_type', type: 'string', defaultValue: null },
    { name: 'hasInnateSpellsFilter', urlKey: 'has_innate_spells', type: 'string', defaultValue: null },
    { name: 'selectedAbilityBonuses', urlKey: 'ability', type: 'stringArray', defaultValue: [] }
  ]
})

export interface RaceFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedSize: string
  selectedSpeedRange: string | null
  selectedParentRace: string
  raceTypeFilter: string | null
  hasInnateSpellsFilter: string | null
  selectedAbilityBonuses: string[]
  filtersOpen: boolean
}
```

**Step 3: Run tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/raceFilters.test.ts && docker compose exec nuxt npm run test:races`

**Step 4: Commit**

```bash
git add app/stores/raceFilters.ts
git commit -m "$(cat <<'EOF'
refactor(stores): Migrate raceFilters to factory pattern

Replaces 168 lines with ~35 lines.
Uses emptyString type for selectedSize and selectedParentRace.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 10: Migrate backgroundFilters to Factory Pattern

**Files:**
- Modify: `app/stores/backgroundFilters.ts`

**Step 1: Run existing tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/backgroundFilters.test.ts`

**Step 2: Rewrite using factory**

```typescript
// app/stores/backgroundFilters.ts
import { createEntityFilterStore } from './filterFactory'
import { STORE_KEYS } from './types'

export const useBackgroundFiltersStore = createEntityFilterStore({
  name: 'backgroundFilters',
  storageKey: STORE_KEYS.backgrounds,
  fields: [
    { name: 'selectedSkills', urlKey: 'skill', type: 'stringArray', defaultValue: [] },
    { name: 'selectedToolTypes', urlKey: 'tool_type', type: 'stringArray', defaultValue: [] },
    { name: 'languageChoiceFilter', urlKey: 'grants_language_choice', type: 'string', defaultValue: null }
  ]
})

export interface BackgroundFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  selectedSkills: string[]
  selectedToolTypes: string[]
  languageChoiceFilter: string | null
  filtersOpen: boolean
}
```

**Step 3: Run tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/backgroundFilters.test.ts && docker compose exec nuxt npm run test:backgrounds`

**Step 4: Commit**

```bash
git add app/stores/backgroundFilters.ts
git commit -m "$(cat <<'EOF'
refactor(stores): Migrate backgroundFilters to factory pattern

Replaces 137 lines with ~30 lines.
Removes 'as any' TypeScript hack that was previously needed.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 11: Migrate featFilters to Factory Pattern

**Files:**
- Modify: `app/stores/featFilters.ts`

**Step 1: Run existing tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/featFilters.test.ts`

**Step 2: Rewrite using factory**

```typescript
// app/stores/featFilters.ts
import { createEntityFilterStore } from './filterFactory'
import { STORE_KEYS } from './types'

export const useFeatFiltersStore = createEntityFilterStore({
  name: 'featFilters',
  storageKey: STORE_KEYS.feats,
  fields: [
    { name: 'hasPrerequisites', urlKey: 'has_prerequisites', type: 'string', defaultValue: null },
    { name: 'grantsProficiencies', urlKey: 'grants_proficiencies', type: 'string', defaultValue: null },
    { name: 'selectedImprovedAbilities', urlKey: 'improved_ability', type: 'stringArray', defaultValue: [] },
    { name: 'selectedPrerequisiteTypes', urlKey: 'prerequisite_type', type: 'stringArray', defaultValue: [] }
  ]
})

export interface FeatFiltersState {
  searchQuery: string
  sortBy: string
  sortDirection: 'asc' | 'desc'
  selectedSources: string[]
  hasPrerequisites: string | null
  grantsProficiencies: string | null
  selectedImprovedAbilities: string[]
  selectedPrerequisiteTypes: string[]
  filtersOpen: boolean
}
```

**Step 3: Run tests**

Run: `docker compose exec nuxt npm run test -- tests/stores/featFilters.test.ts && docker compose exec nuxt npm run test:feats`

**Step 4: Commit**

```bash
git add app/stores/featFilters.ts
git commit -m "$(cat <<'EOF'
refactor(stores): Migrate featFilters to factory pattern

Replaces 148 lines with ~30 lines.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 12: Run Full Test Suite and Verify

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm run test`
Expected: All tests pass (~125s)

**Step 2: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

**Step 3: Run lint**

Run: `docker compose exec nuxt npm run lint`
Expected: No errors (or fix with `npm run lint:fix`)

**Step 4: Verify in browser**

1. Start dev server: `docker compose exec nuxt npm run dev`
2. Open http://localhost:3000/spells
3. Test: Apply filters, reload page (filters should persist)
4. Test: Copy URL with filters, open in new tab (URL sync works)
5. Repeat for /items, /monsters, /classes, /races, /backgrounds, /feats

**Step 5: Final commit (if any fixes needed)**

```bash
git add -A
git commit -m "$(cat <<'EOF'
chore: Fix any issues from full test suite

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 13: Update Documentation

**Files:**
- Modify: `CHANGELOG.md`

**Step 1: Add changelog entry**

Add to the top of CHANGELOG.md:

```markdown
## [Unreleased]

### Changed
- **BREAKING (internal):** Filter stores now use factory pattern (`createEntityFilterStore`)
  - No changes to public API - all stores export same interface
  - Reduced store code from 1,385 lines to ~350 lines (75% reduction)
  - New files: `app/stores/filterFactory/` directory

### Technical
- Added `createEntityFilterStore()` factory function
- Added `FilterFieldDefinition` type system for declarative store configuration
- All 7 entity filter stores migrated to factory pattern
```

**Step 2: Commit**

```bash
git add CHANGELOG.md
git commit -m "$(cat <<'EOF'
docs: Update CHANGELOG for filter store factory refactoring

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Summary

| Metric | Before | After | Reduction |
|--------|--------|-------|-----------|
| spellFilters.ts | 229 lines | ~50 lines | 78% |
| itemFilters.ts | 281 lines | ~55 lines | 80% |
| monsterFilters.ts | 264 lines | ~50 lines | 81% |
| classFilters.ts | 158 lines | ~35 lines | 78% |
| raceFilters.ts | 168 lines | ~35 lines | 79% |
| backgroundFilters.ts | 137 lines | ~30 lines | 78% |
| featFilters.ts | 148 lines | ~30 lines | 80% |
| **Store subtotal** | **1,385 lines** | **~285 lines** | **79%** |
| Factory code (new) | 0 lines | ~250 lines | N/A |
| **Total** | **1,385 lines** | **~535 lines** | **61%** |

**Benefits:**
1. Adding a new filter = 1 line (field definition) vs. 5+ places to edit
2. Consistent behavior guaranteed across all stores
3. Single source of truth for filter logic
4. Easier to test (factory tests cover all stores)
5. TypeScript provides autocompletion for field configs
