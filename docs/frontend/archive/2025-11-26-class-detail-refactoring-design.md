# Class Detail Page Refactoring Design

**Date:** 2025-11-26
**Status:** Approved
**Author:** Claude (AI Assistant)

## Overview

Refactor the frontend class detail page (`app/pages/classes/[slug].vue`) and its components to consume pre-computed backend data instead of performing calculations locally.

## Background

The backend team has implemented computed fields on the `ClassResource` API that eliminate the need for frontend calculations:

- **`computed.hit_points`** - Pre-calculated D&D 5e hit point formulas
- **`computed.progression_table`** - Complete 20-level table with dynamic columns
- **`computed.spell_slot_summary`** - Caster type and spell slot metadata
- **`computed.section_counts`** - Counts for accordion badge labels
- **`inherited_data`** - Pre-resolved parent class data for subclasses

## Goals

1. Delete all frontend calculation logic (~100 lines)
2. Simplify component props to accept pre-computed objects
3. Use generated TypeScript types (no custom types needed)
4. Maintain existing visual appearance and functionality
5. Support both base classes and subclasses

## API Structure

```
ClassResource
├── id, slug, name, hit_die, description, ...
├── parent_class, subclasses, features, ...
├── inherited_data (subclasses only)
│   ├── hit_die, hit_points, counters, traits
│   ├── level_progression, equipment, proficiencies
│   └── spell_slot_summary
└── computed (detail view only)
    ├── hit_points {hit_die, hit_die_numeric, first_level, higher_levels}
    ├── spell_slot_summary {has_spell_slots, max_spell_level, caster_type, ...}
    ├── section_counts {features, proficiencies, traits, subclasses, ...}
    └── progression_table {columns: [...], rows: [...]}
```

## Component Changes

### 1. UiClassHitPointsCard

**Before:**
```typescript
interface Props {
  hitDie: number
  className: string
}
const averageHp = computed(() => Math.floor(props.hitDie / 2) + 1)
```

**After:**
```typescript
interface Props {
  hitPoints: {
    hit_die: string
    hit_die_numeric: number
    first_level: { value: number; description: string }
    higher_levels: { roll: string; average: number; description: string }
  }
}
// No computed logic - render hitPoints.first_level.description directly
```

**Lines deleted:** ~15 lines of calculation logic

### 2. UiClassProgressionTable

**Before (~130 lines):**
- `getProficiencyBonus()` - calculates Math.ceil(level / 4) + 1
- `featuresByLevel` - groups features by level
- `counterNames` - extracts unique counter names
- `getCounterAtLevel()` - interpolates counter values
- `formatCounterValue()` - hardcodes Sneak Attack → d6 format
- `columns` computed - builds column array
- `tableRows` computed - builds 20 rows with calculations

**After (~40 lines):**
```typescript
interface Props {
  progressionTable: {
    columns: { key: string; label: string; type: string }[]
    rows: { [key: string]: unknown }[]
  }
}
// Template iterates columns and rows directly
```

**Lines deleted:** ~90 lines of calculation/interpolation logic

### 3. [slug].vue Page

**Before (~50 lines of inheritance logic):**
```typescript
const progressionFeatures = computed(() => {
  if (isSubclass.value && parentClass.value?.features) {
    return parentClass.value.features.filter(...)
  }
  return baseClassFeatures.value
})
// Similar patterns for: counters, traits, levelProgression, equipment, proficiencies
```

**After:**
```typescript
// For base classes: use entity.computed directly
// For subclasses: use entity.inherited_data for parent's data

const hitPointsData = computed(() =>
  entity.value?.computed?.hit_points ?? null
)

const progressionTableData = computed(() =>
  entity.value?.computed?.progression_table ?? null
)

// Accordion data for subclasses uses inherited_data
const accordionProficiencies = computed(() =>
  entity.value?.is_base_class
    ? entity.value?.proficiencies
    : entity.value?.inherited_data?.proficiencies
)
```

**Lines deleted:** ~30 lines of manual inheritance fallback logic

## Type Usage

Use generated types from `app/types/api/generated.ts`:

```typescript
import type { components } from '~/types/api/generated'

type ClassComputedResource = components['schemas']['ClassComputedResource']
type ClassHitPoints = NonNullable<ClassComputedResource['hit_points']>
type ClassProgressionTable = NonNullable<ClassComputedResource['progression_table']>
```

No custom type definitions required.

## Test Updates

Test mocks need to provide the new data shape:

```typescript
// Old mock
const mockClass = {
  hit_die: 10,
  features: [...],
  counters: [...]
}

// New mock
const mockClass = {
  hit_die: 10,
  computed: {
    hit_points: {
      hit_die: 'd10',
      hit_die_numeric: 10,
      first_level: { value: 10, description: '10 + your Constitution modifier' },
      higher_levels: { roll: '1d10', average: 6, description: '1d10 (or 6) + ...' }
    },
    progression_table: {
      columns: [...],
      rows: [...]
    },
    section_counts: { features: 31, proficiencies: 16, ... },
    spell_slot_summary: null
  },
  inherited_data: null  // or populated for subclass tests
}
```

## Implementation Order

1. **Sync types** - Run `npm run types:sync` (already done)
2. **Update entities.ts** - Add helper type exports for convenience
3. **Refactor UiClassHitPointsCard** - TDD: write tests, then implement
4. **Refactor UiClassProgressionTable** - TDD: write tests, then implement
5. **Update [slug].vue** - Use new props, simplify inheritance logic
6. **Update test mocks** - All tests use new data structure
7. **Manual verification** - Check Fighter, Wizard, Arcane Trickster in browser
8. **Commit** - Single commit with all changes

## Success Criteria

- [ ] All frontend calculation logic removed
- [ ] Components accept pre-computed props
- [ ] All existing tests pass (with updated mocks)
- [ ] Base classes display correctly (Fighter, Wizard)
- [ ] Subclasses display correctly with inherited data (Arcane Trickster)
- [ ] Spell casters show spell slot columns in progression table
- [ ] TypeScript compiles with no errors

## Files Modified

- `app/types/api/entities.ts` - Add type exports
- `app/components/ui/class/UiClassHitPointsCard.vue` - New props, remove calculations
- `app/components/ui/class/UiClassProgressionTable.vue` - New props, remove calculations
- `app/pages/classes/[slug].vue` - Use computed/inherited_data, simplify
- `tests/components/class/*.test.ts` - Update mocks
- `tests/pages/classes/*.test.ts` - Update mocks

## Risks & Mitigations

| Risk | Mitigation |
|------|------------|
| API doesn't return computed for some classes | Check API response, fall back gracefully |
| Test mocks don't match real API | Use actual API responses as mock templates |
| Breaking change missed | Run full test suite before committing |
