# Class Detail Page Backend Data Refactoring - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Refactor class detail page components to consume pre-computed backend data instead of calculating locally.

**Architecture:** Replace frontend calculations with direct consumption of `entity.computed.*` fields. Use `entity.inherited_data.*` for subclass inheritance. Delete ~100 lines of calculation logic.

**Tech Stack:** Vue 3, TypeScript, Vitest, NuxtUI

---

## Task 1: Add Type Exports to entities.ts

**Files:**
- Modify: `app/types/api/entities.ts`

**Step 1: Add ClassComputedResource type exports**

Add these exports after the existing `ClassCounterResource` export (around line 142):

```typescript
/**
 * Class computed data types for detail views
 * These are pre-calculated by the backend to eliminate frontend calculations
 */
export type ClassComputedResource = components['schemas']['ClassComputedResource']

/** Pre-computed hit points with D&D 5e formulas */
export type ClassHitPoints = NonNullable<ClassComputedResource['hit_points']>

/** Pre-computed spell slot summary for caster classes */
export type ClassSpellSlotSummary = NonNullable<ClassComputedResource['spell_slot_summary']>

/** Section counts for lazy-loading accordion badges */
export type ClassSectionCounts = NonNullable<ClassComputedResource['section_counts']>

/** Pre-computed progression table with dynamic columns */
export type ClassProgressionTable = NonNullable<ClassComputedResource['progression_table']>
```

**Step 2: Verify types compile**

Run:
```bash
docker compose exec nuxt npm run typecheck
```

Expected: No new errors

**Step 3: Commit**

```bash
git add app/types/api/entities.ts
git commit -m "chore: Add ClassComputedResource type exports

- Export ClassHitPoints, ClassSpellSlotSummary, ClassSectionCounts
- Export ClassProgressionTable for progression table rendering
- These support the backend computed data refactoring

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Refactor UiClassHitPointsCard - Tests First

**Files:**
- Modify: `tests/components/ui/class/UiClassHitPointsCard.test.ts`

**Step 1: Rewrite tests for new props interface**

Replace the entire test file with:

```typescript
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import UiClassHitPointsCard from '~/components/ui/class/UiClassHitPointsCard.vue'

describe('UiClassHitPointsCard', () => {
  const mountOptions = {
    global: {
      stubs: {
        UCard: {
          template: '<div class="card"><slot /></div>'
        },
        UIcon: {
          template: '<i class="icon" />',
          props: ['name']
        }
      }
    }
  }

  // Mock data matching API response structure
  const createHitPoints = (hitDie: number, className: string) => ({
    hit_die: `d${hitDie}`,
    hit_die_numeric: hitDie,
    first_level: {
      value: hitDie,
      description: `${hitDie} + your Constitution modifier`
    },
    higher_levels: {
      roll: `1d${hitDie}`,
      average: Math.floor(hitDie / 2) + 1,
      description: `1d${hitDie} (or ${Math.floor(hitDie / 2) + 1}) + your Constitution modifier per ${className} level after 1st`
    }
  })

  it('renders hit die value from backend data', () => {
    const wrapper = mount(UiClassHitPointsCard, {
      props: {
        hitPoints: createHitPoints(8, 'rogue')
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('d8')
  })

  it('renders HP at 1st level from backend description', () => {
    const wrapper = mount(UiClassHitPointsCard, {
      props: {
        hitPoints: createHitPoints(10, 'fighter')
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('10 + your Constitution modifier')
  })

  it('renders HP at higher levels from backend description', () => {
    const wrapper = mount(UiClassHitPointsCard, {
      props: {
        hitPoints: createHitPoints(12, 'barbarian')
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('1d12 (or 7) + your Constitution modifier per barbarian level after 1st')
  })

  it('displays different hit dice correctly', () => {
    // d6 wizard
    const d6Wrapper = mount(UiClassHitPointsCard, {
      props: { hitPoints: createHitPoints(6, 'wizard') },
      ...mountOptions
    })
    expect(d6Wrapper.text()).toContain('d6')
    expect(d6Wrapper.text()).toContain('(or 4)')

    // d8 rogue
    const d8Wrapper = mount(UiClassHitPointsCard, {
      props: { hitPoints: createHitPoints(8, 'rogue') },
      ...mountOptions
    })
    expect(d8Wrapper.text()).toContain('d8')
    expect(d8Wrapper.text()).toContain('(or 5)')

    // d10 fighter
    const d10Wrapper = mount(UiClassHitPointsCard, {
      props: { hitPoints: createHitPoints(10, 'fighter') },
      ...mountOptions
    })
    expect(d10Wrapper.text()).toContain('d10')
    expect(d10Wrapper.text()).toContain('(or 6)')

    // d12 barbarian
    const d12Wrapper = mount(UiClassHitPointsCard, {
      props: { hitPoints: createHitPoints(12, 'barbarian') },
      ...mountOptions
    })
    expect(d12Wrapper.text()).toContain('d12')
    expect(d12Wrapper.text()).toContain('(or 7)')
  })

  it('displays heart icon', () => {
    const wrapper = mount(UiClassHitPointsCard, {
      props: {
        hitPoints: createHitPoints(8, 'rogue')
      },
      ...mountOptions
    })

    expect(wrapper.find('.icon').exists()).toBe(true)
  })
})
```

**Step 2: Run tests to verify they fail**

Run:
```bash
docker compose exec nuxt npm run test -- tests/components/ui/class/UiClassHitPointsCard.test.ts
```

Expected: FAIL - Component still expects old `hitDie` and `className` props

---

## Task 3: Refactor UiClassHitPointsCard - Implementation

**Files:**
- Modify: `app/components/ui/class/UiClassHitPointsCard.vue`

**Step 1: Replace component implementation**

Replace entire file with:

```vue
<script setup lang="ts">
import type { ClassHitPoints } from '~/types/api/entities'

interface Props {
  hitPoints: ClassHitPoints
}

defineProps<Props>()
</script>

<template>
  <UCard>
    <div class="flex items-start gap-4">
      <div class="flex-shrink-0">
        <div class="w-12 h-12 rounded-lg bg-error-100 dark:bg-error-900/30 flex items-center justify-center">
          <UIcon
            name="i-heroicons-heart"
            class="w-6 h-6 text-error-600 dark:text-error-400"
          />
        </div>
      </div>

      <div class="flex-1 space-y-3">
        <h3 class="text-lg font-semibold text-gray-900 dark:text-gray-100">
          Hit Points
        </h3>

        <dl class="space-y-2 text-sm">
          <div class="flex flex-col sm:flex-row sm:gap-2">
            <dt class="font-medium text-gray-600 dark:text-gray-400 sm:w-32">
              Hit Dice
            </dt>
            <dd class="text-gray-900 dark:text-gray-100">
              1{{ hitPoints.hit_die }} per level
            </dd>
          </div>

          <div class="flex flex-col sm:flex-row sm:gap-2">
            <dt class="font-medium text-gray-600 dark:text-gray-400 sm:w-32">
              HP at 1st Level
            </dt>
            <dd class="text-gray-900 dark:text-gray-100">
              {{ hitPoints.first_level.description }}
            </dd>
          </div>

          <div class="flex flex-col sm:flex-row sm:gap-2">
            <dt class="font-medium text-gray-600 dark:text-gray-400 sm:w-32">
              HP at Higher Levels
            </dt>
            <dd class="text-gray-900 dark:text-gray-100">
              {{ hitPoints.higher_levels.description }}
            </dd>
          </div>
        </dl>
      </div>
    </div>
  </UCard>
</template>
```

**Step 2: Run tests to verify they pass**

Run:
```bash
docker compose exec nuxt npm run test -- tests/components/ui/class/UiClassHitPointsCard.test.ts
```

Expected: All 5 tests PASS

**Step 3: Commit**

```bash
git add app/components/ui/class/UiClassHitPointsCard.vue tests/components/ui/class/UiClassHitPointsCard.test.ts
git commit -m "refactor(UiClassHitPointsCard): Use backend computed hit_points

- Replace hitDie/className props with hitPoints object
- Remove frontend calculation logic (averageHp, formattedClassName)
- Render backend-provided descriptions directly
- Update tests to use new props interface

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Refactor UiClassProgressionTable - Tests First

**Files:**
- Modify: `tests/components/ui/class/UiClassProgressionTable.test.ts`

**Step 1: Rewrite tests for new props interface**

Replace entire test file with:

```typescript
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import UiClassProgressionTable from '~/components/ui/class/UiClassProgressionTable.vue'

describe('UiClassProgressionTable', () => {
  const mountOptions = {
    global: {
      stubs: {
        UCard: {
          template: '<div class="card"><template v-if="$slots.header"><div class="card-header"><slot name="header" /></div></template><slot /></div>'
        }
      }
    }
  }

  // Mock progression table matching API response structure
  const mockProgressionTable = {
    columns: [
      { key: 'level', label: 'Level', type: 'integer' },
      { key: 'proficiency_bonus', label: 'Proficiency Bonus', type: 'bonus' },
      { key: 'features', label: 'Features', type: 'string' },
      { key: 'sneak_attack', label: 'Sneak Attack', type: 'dice' }
    ],
    rows: [
      { level: 1, proficiency_bonus: '+2', features: 'Expertise, Sneak Attack', sneak_attack: '1d6' },
      { level: 2, proficiency_bonus: '+2', features: 'Cunning Action', sneak_attack: '1d6' },
      { level: 3, proficiency_bonus: '+2', features: 'Roguish Archetype', sneak_attack: '2d6' },
      { level: 4, proficiency_bonus: '+2', features: 'Ability Score Improvement', sneak_attack: '2d6' },
      { level: 5, proficiency_bonus: '+3', features: 'Uncanny Dodge', sneak_attack: '3d6' }
    ]
  }

  const mockProgressionTableNoCounters = {
    columns: [
      { key: 'level', label: 'Level', type: 'integer' },
      { key: 'proficiency_bonus', label: 'Proficiency Bonus', type: 'bonus' },
      { key: 'features', label: 'Features', type: 'string' }
    ],
    rows: [
      { level: 1, proficiency_bonus: '+2', features: 'Starting Feature' },
      { level: 2, proficiency_bonus: '+2', features: '—' },
      { level: 3, proficiency_bonus: '+2', features: 'Archetype Feature' }
    ]
  }

  it('renders a table with headers from columns', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: { progressionTable: mockProgressionTable },
      ...mountOptions
    })

    expect(wrapper.find('table').exists()).toBe(true)
    expect(wrapper.text()).toContain('Level')
    expect(wrapper.text()).toContain('Proficiency Bonus')
    expect(wrapper.text()).toContain('Features')
    expect(wrapper.text()).toContain('Sneak Attack')
  })

  it('renders all rows from backend data', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: { progressionTable: mockProgressionTable },
      ...mountOptions
    })

    // Should have 5 data rows
    const rows = wrapper.findAll('tbody tr')
    expect(rows.length).toBe(5)
  })

  it('displays features from backend data', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: { progressionTable: mockProgressionTable },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('Expertise, Sneak Attack')
    expect(wrapper.text()).toContain('Cunning Action')
    expect(wrapper.text()).toContain('Roguish Archetype')
  })

  it('displays proficiency bonus from backend data', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: { progressionTable: mockProgressionTable },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('+2')
    expect(wrapper.text()).toContain('+3')
  })

  it('displays counter columns from backend data', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: { progressionTable: mockProgressionTable },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('1d6')
    expect(wrapper.text()).toContain('2d6')
    expect(wrapper.text()).toContain('3d6')
  })

  it('handles tables without counter columns', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: { progressionTable: mockProgressionTableNoCounters },
      ...mountOptions
    })

    expect(wrapper.find('table').exists()).toBe(true)
    expect(wrapper.text()).not.toContain('Sneak Attack')
  })

  it('displays dash for empty feature levels', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: { progressionTable: mockProgressionTableNoCounters },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('—')
  })

  it('renders card header with title', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: { progressionTable: mockProgressionTable },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('Class Progression')
  })
})
```

**Step 2: Run tests to verify they fail**

Run:
```bash
docker compose exec nuxt npm run test -- tests/components/ui/class/UiClassProgressionTable.test.ts
```

Expected: FAIL - Component still expects old `features` and `counters` props

---

## Task 5: Refactor UiClassProgressionTable - Implementation

**Files:**
- Modify: `app/components/ui/class/UiClassProgressionTable.vue`

**Step 1: Replace component implementation**

Replace entire file with:

```vue
<script setup lang="ts">
import type { ClassProgressionTable } from '~/types/api/entities'

interface Props {
  progressionTable: ClassProgressionTable
}

defineProps<Props>()
</script>

<template>
  <UCard>
    <template #header>
      <h3 class="text-lg font-semibold text-gray-900 dark:text-gray-100">
        Class Progression
      </h3>
    </template>

    <div class="overflow-x-auto">
      <table class="w-full text-sm">
        <caption class="sr-only">
          Class progression from level 1 to 20
        </caption>
        <thead class="bg-gray-50 dark:bg-gray-800">
          <tr>
            <th
              v-for="col in progressionTable.columns"
              :key="col.key"
              scope="col"
              class="px-3 py-2 text-left font-semibold text-gray-700 dark:text-gray-300"
              :class="{ 'text-center': col.key === 'level' || col.key === 'proficiency_bonus' || col.type === 'integer' || col.type === 'bonus' || col.type === 'dice' }"
            >
              {{ col.label }}
            </th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="(row, index) in progressionTable.rows"
            :key="row.level"
            class="border-t border-gray-200 dark:border-gray-700"
            :class="{ 'bg-gray-50/50 dark:bg-gray-800/50': index % 2 === 1 }"
          >
            <td
              v-for="col in progressionTable.columns"
              :key="col.key"
              class="px-3 py-2"
              :class="{
                'text-center font-medium text-gray-900 dark:text-gray-100': col.key === 'level',
                'text-center text-gray-700 dark:text-gray-300': col.key !== 'level' && col.key !== 'features',
                'text-gray-700 dark:text-gray-300': col.key === 'features'
              }"
            >
              {{ row[col.key] }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </UCard>
</template>
```

**Step 2: Run tests to verify they pass**

Run:
```bash
docker compose exec nuxt npm run test -- tests/components/ui/class/UiClassProgressionTable.test.ts
```

Expected: All 8 tests PASS

**Step 3: Commit**

```bash
git add app/components/ui/class/UiClassProgressionTable.vue tests/components/ui/class/UiClassProgressionTable.test.ts
git commit -m "refactor(UiClassProgressionTable): Use backend computed progression_table

- Replace features/counters props with progressionTable object
- Remove ~90 lines of calculation logic:
  - getProficiencyBonus() calculation
  - featuresByLevel grouping
  - counterNames extraction
  - getCounterAtLevel() interpolation
  - formatCounterValue() with hardcoded Sneak Attack
- Render dynamic columns and rows directly from backend
- Update tests to use new props interface

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Update [slug].vue Page

**Files:**
- Modify: `app/pages/classes/[slug].vue`

**Step 1: Update imports and computed properties**

In the `<script setup>` section, replace the existing computed properties for progression and hit points:

Find lines ~33-75 (the baseClassFeatures, progressionFeatures, progressionCounters computeds) and replace with:

```typescript
/**
 * Get computed hit points data (works for both base and subclasses)
 */
const hitPointsData = computed(() => {
  return entity.value?.computed?.hit_points ?? null
})

/**
 * Get computed progression table (works for both base and subclasses)
 */
const progressionTableData = computed(() => {
  return entity.value?.computed?.progression_table ?? null
})

/**
 * Determine if viewing a subclass (not a base class)
 */
const isSubclass = computed(() => entity.value && !entity.value.is_base_class)

/**
 * Get parent class data (only available for subclasses)
 */
const parentClass = computed(() => entity.value?.parent_class)
```

**Step 2: Update accordionItems to use inherited_data**

Replace the accordionItems computed (around lines 81-187) to use `entity.inherited_data` for subclasses:

```typescript
/**
 * Build accordion items with icons
 * For subclasses, uses inherited_data from backend
 */
const accordionItems = computed(() => {
  if (!entity.value) return []

  const items = []
  const isBase = entity.value.is_base_class

  // For subclasses, use inherited_data; for base classes, use entity directly
  const counters = isBase
    ? entity.value.counters
    : entity.value.inherited_data?.counters

  const traits = isBase
    ? entity.value.traits
    : entity.value.inherited_data?.traits

  const levelProgression = isBase
    ? entity.value.level_progression
    : entity.value.inherited_data?.level_progression

  const equipment = isBase
    ? entity.value.equipment
    : entity.value.inherited_data?.equipment

  const proficiencies = isBase
    ? entity.value.proficiencies
    : entity.value.inherited_data?.proficiencies

  // Features are always from the entity itself (subclasses have their own features)
  const features = entity.value.features

  const inheritedLabel = isSubclass.value && parentClass.value
    ? ` (Inherited from ${parentClass.value.name})`
    : ''

  if (counters && counters.length > 0) {
    items.push({
      label: `Class Counters${inheritedLabel}`,
      slot: 'counters',
      defaultOpen: false,
      icon: 'i-heroicons-calculator'
    })
  }

  if (traits && traits.length > 0) {
    items.push({
      label: `Class Traits (${traits.length})${inheritedLabel}`,
      slot: 'traits',
      defaultOpen: false,
      icon: 'i-heroicons-shield-check'
    })
  }

  if (levelProgression && levelProgression.length > 0) {
    items.push({
      label: `Spell Slot Progression${inheritedLabel}`,
      slot: 'level-progression',
      defaultOpen: false,
      icon: 'i-heroicons-sparkles'
    })
  }

  if (equipment && equipment.length > 0) {
    items.push({
      label: `Starting Equipment${inheritedLabel}`,
      slot: 'equipment',
      defaultOpen: false,
      icon: 'i-heroicons-shopping-bag'
    })
  }

  if (proficiencies && proficiencies.length > 0) {
    items.push({
      label: `Proficiencies (${proficiencies.length})${inheritedLabel}`,
      slot: 'proficiencies',
      defaultOpen: false,
      icon: 'i-heroicons-academic-cap'
    })
  }

  // Features section - base class shows all, subclass shows own features
  if (features && features.length > 0) {
    items.push({
      label: `Features (${features.length})`,
      slot: 'features',
      defaultOpen: false,
      icon: 'i-heroicons-star'
    })
  }

  // Source is always from the current entity
  if (entity.value.sources && entity.value.sources.length > 0) {
    items.push({
      label: 'Source',
      slot: 'source',
      defaultOpen: false,
      icon: 'i-heroicons-book-open'
    })
  }

  return items
})

/**
 * Get data for accordion slots (handles inheritance via inherited_data)
 */
const accordionData = computed(() => {
  if (!entity.value) return {}

  const isBase = entity.value.is_base_class

  return {
    counters: isBase ? entity.value.counters : entity.value.inherited_data?.counters,
    traits: isBase ? entity.value.traits : entity.value.inherited_data?.traits,
    levelProgression: isBase ? entity.value.level_progression : entity.value.inherited_data?.level_progression,
    equipment: isBase ? entity.value.equipment : entity.value.inherited_data?.equipment,
    proficiencies: isBase ? entity.value.proficiencies : entity.value.inherited_data?.proficiencies,
    features: entity.value.features
  }
})
```

**Step 3: Update template to use new computed properties**

Find the Class Progression Table section (around line 334-362) and replace with:

```vue
<!-- Class Progression Table (from computed data) -->
<div
  v-if="progressionTableData"
  class="space-y-2"
>
  <div
    v-if="isSubclass && parentClass"
    class="flex items-center gap-2"
  >
    <h3 class="text-lg font-semibold text-gray-900 dark:text-gray-100 flex items-center gap-2">
      <UIcon
        name="i-heroicons-table-cells"
        class="w-5 h-5 text-gray-400"
      />
      Class Progression
    </h3>
    <UBadge
      color="neutral"
      variant="subtle"
      size="xs"
    >
      Inherited from {{ parentClass.name }}
    </UBadge>
  </div>
  <UiClassProgressionTable
    :progression-table="progressionTableData"
  />
</div>
```

Find the Hit Points Card section (around line 370-401) and replace with:

```vue
<!-- Hit Points Card (from computed data) -->
<UiClassHitPointsCard
  v-if="hitPointsData"
  :hit-points="hitPointsData"
/>
```

**Step 4: Run class domain tests**

Run:
```bash
docker compose exec nuxt npm run test:classes
```

Expected: All tests pass (may need mock updates - see Task 7)

**Step 5: Commit**

```bash
git add app/pages/classes/[slug].vue
git commit -m "refactor(classes/[slug]): Use backend computed and inherited_data

- Use entity.computed.hit_points for hit points card
- Use entity.computed.progression_table for progression table
- Use entity.inherited_data for subclass inheritance instead of parent_class
- Remove manual inheritance fallback logic
- Simplify accordionItems and accordionData computed properties

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Update Test Mocks (if needed)

**Files:**
- Modify: Any failing test files

**Step 1: Run full class test suite**

Run:
```bash
docker compose exec nuxt npm run test:classes
```

**Step 2: Update mocks in failing tests**

For any tests that fail, update mocks to include `computed` and `inherited_data` fields:

```typescript
const mockBaseClass = {
  id: 1,
  name: 'Fighter',
  slug: 'fighter',
  hit_die: 10,
  is_base_class: true,
  computed: {
    hit_points: {
      hit_die: 'd10',
      hit_die_numeric: 10,
      first_level: { value: 10, description: '10 + your Constitution modifier' },
      higher_levels: { roll: '1d10', average: 6, description: '1d10 (or 6) + your Constitution modifier per fighter level after 1st' }
    },
    progression_table: {
      columns: [
        { key: 'level', label: 'Level', type: 'integer' },
        { key: 'proficiency_bonus', label: 'Proficiency Bonus', type: 'bonus' },
        { key: 'features', label: 'Features', type: 'string' }
      ],
      rows: [
        { level: 1, proficiency_bonus: '+2', features: 'Fighting Style, Second Wind' }
      ]
    },
    section_counts: { features: 10, proficiencies: 5, traits: 2, subclasses: 3, spells: 0, counters: 2, optional_features: 0 },
    spell_slot_summary: null
  },
  inherited_data: null
}

const mockSubclass = {
  id: 2,
  name: 'Champion',
  slug: 'fighter-champion',
  hit_die: 10,
  is_base_class: false,
  parent_class_id: 1,
  computed: {
    hit_points: {
      hit_die: 'd10',
      hit_die_numeric: 10,
      first_level: { value: 10, description: '10 + your Constitution modifier' },
      higher_levels: { roll: '1d10', average: 6, description: '1d10 (or 6) + your Constitution modifier per champion level after 1st' }
    },
    progression_table: { columns: [...], rows: [...] },
    section_counts: { features: 5, proficiencies: 0, traits: 0, subclasses: 0, spells: 0, counters: 0, optional_features: 0 },
    spell_slot_summary: null
  },
  inherited_data: {
    hit_die: 10,
    hit_points: { ... },
    counters: [...],
    traits: [...],
    level_progression: [...],
    equipment: [...],
    proficiencies: [...],
    spell_slot_summary: null
  }
}
```

**Step 3: Commit any mock updates**

```bash
git add tests/
git commit -m "test: Update class test mocks for computed/inherited_data

- Add computed field with hit_points, progression_table, section_counts
- Add inherited_data for subclass tests
- Mocks now match actual API response structure

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 8: Manual Verification

**Step 1: Start dev server**

Run:
```bash
docker compose exec nuxt npm run dev
```

**Step 2: Test base class (Fighter)**

Open: http://localhost:3000/classes/fighter

Verify:
- [ ] Hit Points card displays correctly
- [ ] Progression table shows 20 levels
- [ ] Progression table has Action Surge, Indomitable, Second Wind columns
- [ ] Features display in progression table
- [ ] Proficiency bonuses are correct (+2 at level 1, +6 at level 17)

**Step 3: Test spellcaster class (Wizard)**

Open: http://localhost:3000/classes/wizard

Verify:
- [ ] Hit Points card shows d6
- [ ] Progression table has spell slot columns (1st through 9th)
- [ ] Cantrips Known column visible
- [ ] Arcane Recovery counter column visible

**Step 4: Test subclass (Arcane Trickster)**

Open: http://localhost:3000/classes/rogue-arcane-trickster

Verify:
- [ ] Hit Points card shows d8 (inherited from Rogue)
- [ ] Progression table shows correct progression
- [ ] "Inherited from Rogue" badge appears where appropriate
- [ ] Subclass features display correctly

**Step 5: Run full test suite**

Run:
```bash
docker compose exec nuxt npm run test
```

Expected: All tests pass

---

## Task 9: Final Commit and Cleanup

**Step 1: Run typecheck**

Run:
```bash
docker compose exec nuxt npm run typecheck
```

Expected: No errors

**Step 2: Run lint**

Run:
```bash
docker compose exec nuxt npm run lint
```

Expected: No errors (or fix any that appear)

**Step 3: Commit generated types**

```bash
git add app/types/api/generated.ts
git commit -m "chore: Sync API types with backend ClassComputedResource

- ClassResource now includes computed and inherited_data fields
- ClassComputedResource schema with hit_points, progression_table, etc.
- All nested types properly structured

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

**Step 4: Update CHANGELOG.md**

Add to CHANGELOG.md:

```markdown
### Changed
- Class detail page now uses backend-computed data (2025-11-26)
  - Hit points calculated server-side
  - Progression table with dynamic columns from backend
  - Subclass inheritance resolved by API
```

**Step 5: Final commit**

```bash
git add CHANGELOG.md
git commit -m "docs: Add class refactoring to CHANGELOG

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Summary

| Task | Files Changed | Lines Removed | Lines Added |
|------|--------------|---------------|-------------|
| 1. Type exports | entities.ts | 0 | ~15 |
| 2-3. UiClassHitPointsCard | component + test | ~25 | ~45 |
| 4-5. UiClassProgressionTable | component + test | ~130 | ~75 |
| 6. [slug].vue | page | ~50 | ~40 |
| 7. Test mocks | various tests | varies | varies |

**Total: ~200 lines removed, ~175 lines added = ~25 lines net reduction**

More importantly: **All calculation logic moved to backend** - no more frontend math for proficiency bonuses, hit point averages, or counter interpolation.
