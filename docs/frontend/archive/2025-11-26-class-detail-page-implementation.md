# Class Detail Page Enhancement - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Enhance the Classes detail page with a class progression table, icons on accordion headers, hit points card, and improved subclass display.

**Architecture:** Add 4 new components to `app/components/ui/class/`, enhance the existing page at `app/pages/classes/[slug].vue`. Each component follows existing patterns from `UiDetailQuickStatsCard` and accordion components. TDD approach with tests written first.

**Tech Stack:** Vue 3 + Nuxt 4, NuxtUI 4 (UCard, UTable, UBadge, UIcon), TypeScript, Vitest for testing.

---

## Task 1: UiClassProgressionTable Component - Test Setup

**Files:**
- Create: `tests/components/ui/class/UiClassProgressionTable.test.ts`

**Step 1: Write the failing test file**

```typescript
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import UiClassProgressionTable from '~/components/ui/class/UiClassProgressionTable.vue'

describe('UiClassProgressionTable', () => {
  const mountOptions = {
    global: {
      stubs: {
        UCard: {
          template: '<div class="card"><slot /></div>'
        },
        UTable: {
          template: '<table class="table"><slot /></table>',
          props: ['data', 'columns']
        },
        UBadge: {
          template: '<span class="badge"><slot /></span>',
          props: ['color', 'variant', 'size']
        }
      }
    }
  }

  const mockFeatures = [
    { id: 1, level: 1, feature_name: 'Expertise', description: 'Choose two proficiencies', is_optional: false, sort_order: 1 },
    { id: 2, level: 1, feature_name: 'Sneak Attack', description: 'Deal extra damage', is_optional: false, sort_order: 2 },
    { id: 3, level: 1, feature_name: "Thieves' Cant", description: 'Secret language', is_optional: false, sort_order: 3 },
    { id: 4, level: 2, feature_name: 'Cunning Action', description: 'Bonus action Dash/Disengage/Hide', is_optional: false, sort_order: 4 },
    { id: 5, level: 3, feature_name: 'Roguish Archetype', description: 'Choose subclass', is_optional: false, sort_order: 5 }
  ]

  const mockCounters = [
    { id: 1, level: 1, counter_name: 'Sneak Attack', counter_value: 1, reset_timing: 'Does Not Reset' as const },
    { id: 2, level: 3, counter_name: 'Sneak Attack', counter_value: 2, reset_timing: 'Does Not Reset' as const },
    { id: 3, level: 5, counter_name: 'Sneak Attack', counter_value: 3, reset_timing: 'Does Not Reset' as const }
  ]

  it('renders a table with level column', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: {
        features: mockFeatures,
        counters: []
      },
      ...mountOptions
    })

    expect(wrapper.find('.table').exists()).toBe(true)
  })

  it('renders all 20 levels by default', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: {
        features: mockFeatures,
        counters: []
      },
      ...mountOptions
    })

    // Component should generate 20 rows
    expect(wrapper.vm.tableRows.length).toBe(20)
  })

  it('groups features by level', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: {
        features: mockFeatures,
        counters: []
      },
      ...mountOptions
    })

    const level1Row = wrapper.vm.tableRows.find((r: { level: number }) => r.level === 1)
    expect(level1Row.features).toContain('Expertise')
    expect(level1Row.features).toContain('Sneak Attack')
    expect(level1Row.features).toContain("Thieves' Cant")
  })

  it('calculates proficiency bonus correctly', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: {
        features: [],
        counters: []
      },
      ...mountOptions
    })

    const rows = wrapper.vm.tableRows
    expect(rows[0].proficiencyBonus).toBe('+2')  // Level 1
    expect(rows[4].proficiencyBonus).toBe('+3')  // Level 5
    expect(rows[8].proficiencyBonus).toBe('+4')  // Level 9
    expect(rows[12].proficiencyBonus).toBe('+5') // Level 13
    expect(rows[16].proficiencyBonus).toBe('+6') // Level 17
  })

  it('adds counter columns dynamically', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: {
        features: [],
        counters: mockCounters
      },
      ...mountOptions
    })

    expect(wrapper.vm.columns.some((c: { key: string }) => c.key === 'counter_Sneak Attack')).toBe(true)
  })

  it('interpolates counter values for levels without explicit entries', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: {
        features: [],
        counters: mockCounters
      },
      ...mountOptions
    })

    const rows = wrapper.vm.tableRows
    // Level 1 has explicit value 1
    expect(rows[0]['counter_Sneak Attack']).toBe('1d6')
    // Level 2 should carry forward value 1
    expect(rows[1]['counter_Sneak Attack']).toBe('1d6')
    // Level 3 has explicit value 2
    expect(rows[2]['counter_Sneak Attack']).toBe('2d6')
    // Level 4 should carry forward value 2
    expect(rows[3]['counter_Sneak Attack']).toBe('2d6')
  })

  it('shows dash for empty feature levels', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: {
        features: mockFeatures,
        counters: []
      },
      ...mountOptions
    })

    const level4Row = wrapper.vm.tableRows.find((r: { level: number }) => r.level === 4)
    expect(level4Row.features).toBe('—')
  })

  it('does not render when features array is empty and no counters', () => {
    const wrapper = mount(UiClassProgressionTable, {
      props: {
        features: [],
        counters: []
      },
      ...mountOptions
    })

    // Should still render the table structure (empty progression is valid)
    expect(wrapper.find('.card').exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/ui/class/UiClassProgressionTable.test.ts`

Expected: FAIL with "Cannot find module" or similar (component doesn't exist yet)

**Step 3: Commit test file**

```bash
git add tests/components/ui/class/UiClassProgressionTable.test.ts
git commit -m "test: Add UiClassProgressionTable test scaffolding (RED)"
```

---

## Task 2: UiClassProgressionTable Component - Implementation

**Files:**
- Create: `app/components/ui/class/UiClassProgressionTable.vue`

**Step 1: Write minimal implementation to pass tests**

```vue
<script setup lang="ts">
import type { components } from '~/types/api/generated'

type ClassFeature = components['schemas']['ClassFeatureResource']
type ClassCounter = components['schemas']['ClassCounterResource']

interface Props {
  features: ClassFeature[]
  counters: ClassCounter[]
  maxLevel?: number
}

const props = withDefaults(defineProps<Props>(), {
  maxLevel: 20
})

/**
 * Calculate proficiency bonus for a given level
 * Formula: Math.ceil(level / 4) + 1
 */
const getProficiencyBonus = (level: number): string => {
  const bonus = Math.ceil(level / 4) + 1
  return `+${bonus}`
}

/**
 * Group features by level
 */
const featuresByLevel = computed(() => {
  const grouped = new Map<number, string[]>()

  for (const feature of props.features) {
    const level = feature.level
    const names = grouped.get(level) || []
    names.push(feature.feature_name)
    grouped.set(level, names)
  }

  return grouped
})

/**
 * Get unique counter names to create dynamic columns
 */
const counterNames = computed(() => {
  const names = new Set<string>()
  for (const counter of props.counters) {
    names.add(counter.counter_name)
  }
  return Array.from(names)
})

/**
 * Get counter value at a specific level (find highest entry at or below level)
 */
const getCounterAtLevel = (counterName: string, level: number): number | null => {
  const entries = props.counters
    .filter(c => c.counter_name === counterName && c.level <= level)
    .sort((a, b) => b.level - a.level)

  return entries[0]?.counter_value ?? null
}

/**
 * Format counter value for display (e.g., 1 -> "1d6" for Sneak Attack)
 */
const formatCounterValue = (counterName: string, value: number | null): string => {
  if (value === null) return '—'

  // Sneak Attack uses d6 dice
  if (counterName === 'Sneak Attack') {
    return `${value}d6`
  }

  // Default: just show the number
  return String(value)
}

/**
 * Build dynamic columns array
 */
const columns = computed(() => {
  const cols = [
    { key: 'level', label: 'Level' },
    { key: 'proficiencyBonus', label: 'Prof. Bonus' },
    { key: 'features', label: 'Features' }
  ]

  // Add counter columns
  for (const name of counterNames.value) {
    cols.push({
      key: `counter_${name}`,
      label: name
    })
  }

  return cols
})

/**
 * Build table rows for all levels
 */
const tableRows = computed(() => {
  const rows = []

  for (let level = 1; level <= props.maxLevel; level++) {
    const row: Record<string, unknown> = {
      level,
      proficiencyBonus: getProficiencyBonus(level),
      features: featuresByLevel.value.get(level)?.join(', ') || '—'
    }

    // Add counter values
    for (const name of counterNames.value) {
      const value = getCounterAtLevel(name, level)
      row[`counter_${name}`] = formatCounterValue(name, value)
    }

    rows.push(row)
  }

  return rows
})
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
        <thead class="bg-gray-50 dark:bg-gray-800">
          <tr>
            <th
              v-for="col in columns"
              :key="col.key"
              class="px-3 py-2 text-left font-semibold text-gray-700 dark:text-gray-300"
              :class="{ 'text-center': col.key === 'level' || col.key === 'proficiencyBonus' || col.key.startsWith('counter_') }"
            >
              {{ col.label }}
            </th>
          </tr>
        </thead>
        <tbody>
          <tr
            v-for="row in tableRows"
            :key="row.level"
            class="border-t border-gray-200 dark:border-gray-700"
            :class="{ 'bg-gray-50/50 dark:bg-gray-800/50': (row.level as number) % 2 === 0 }"
          >
            <td class="px-3 py-2 text-center font-medium text-gray-900 dark:text-gray-100">
              {{ row.level }}
            </td>
            <td class="px-3 py-2 text-center text-gray-700 dark:text-gray-300">
              {{ row.proficiencyBonus }}
            </td>
            <td class="px-3 py-2 text-gray-700 dark:text-gray-300">
              {{ row.features }}
            </td>
            <td
              v-for="name in counterNames"
              :key="name"
              class="px-3 py-2 text-center text-gray-700 dark:text-gray-300"
            >
              {{ row[`counter_${name}`] }}
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </UCard>
</template>
```

**Step 2: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/ui/class/UiClassProgressionTable.test.ts`

Expected: All tests PASS

**Step 3: Commit implementation**

```bash
git add app/components/ui/class/UiClassProgressionTable.vue
git commit -m "feat: Add UiClassProgressionTable component (GREEN)"
```

---

## Task 3: UiHitPointsCard Component - Test Setup

**Files:**
- Create: `tests/components/ui/class/UiHitPointsCard.test.ts`

**Step 1: Write the failing test file**

```typescript
import { describe, it, expect } from 'vitest'
import { mount } from '@vue/test-utils'
import UiHitPointsCard from '~/components/ui/class/UiHitPointsCard.vue'

describe('UiHitPointsCard', () => {
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

  it('renders hit die value', () => {
    const wrapper = mount(UiHitPointsCard, {
      props: {
        hitDie: 8,
        className: 'rogue'
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('1d8')
  })

  it('renders HP at 1st level correctly', () => {
    const wrapper = mount(UiHitPointsCard, {
      props: {
        hitDie: 10,
        className: 'fighter'
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('10 + Constitution modifier')
  })

  it('renders HP at higher levels correctly', () => {
    const wrapper = mount(UiHitPointsCard, {
      props: {
        hitDie: 12,
        className: 'barbarian'
      },
      ...mountOptions
    })

    // Should show both die roll and average
    expect(wrapper.text()).toContain('1d12')
    expect(wrapper.text()).toContain('(or 7)')
  })

  it('calculates average HP correctly for different hit dice', () => {
    // d6 average = 4
    const d6Wrapper = mount(UiHitPointsCard, {
      props: { hitDie: 6, className: 'wizard' },
      ...mountOptions
    })
    expect(d6Wrapper.text()).toContain('(or 4)')

    // d8 average = 5
    const d8Wrapper = mount(UiHitPointsCard, {
      props: { hitDie: 8, className: 'rogue' },
      ...mountOptions
    })
    expect(d8Wrapper.text()).toContain('(or 5)')

    // d10 average = 6
    const d10Wrapper = mount(UiHitPointsCard, {
      props: { hitDie: 10, className: 'fighter' },
      ...mountOptions
    })
    expect(d10Wrapper.text()).toContain('(or 6)')

    // d12 average = 7
    const d12Wrapper = mount(UiHitPointsCard, {
      props: { hitDie: 12, className: 'barbarian' },
      ...mountOptions
    })
    expect(d12Wrapper.text()).toContain('(or 7)')
  })

  it('includes class name in per-level description', () => {
    const wrapper = mount(UiHitPointsCard, {
      props: {
        hitDie: 8,
        className: 'rogue'
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('per rogue level')
  })

  it('displays heart icon', () => {
    const wrapper = mount(UiHitPointsCard, {
      props: {
        hitDie: 8,
        className: 'rogue'
      },
      ...mountOptions
    })

    expect(wrapper.find('.icon').exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/ui/class/UiHitPointsCard.test.ts`

Expected: FAIL (component doesn't exist)

**Step 3: Commit test file**

```bash
git add tests/components/ui/class/UiHitPointsCard.test.ts
git commit -m "test: Add UiHitPointsCard test scaffolding (RED)"
```

---

## Task 4: UiHitPointsCard Component - Implementation

**Files:**
- Create: `app/components/ui/class/UiHitPointsCard.vue`

**Step 1: Write minimal implementation**

```vue
<script setup lang="ts">
interface Props {
  hitDie: number
  className: string
}

const props = defineProps<Props>()

/**
 * Calculate average HP per level (rounded up as per D&D rules)
 * Formula: (hitDie / 2) + 1
 */
const averageHp = computed(() => {
  return Math.floor(props.hitDie / 2) + 1
})

/**
 * Format class name for display (lowercase for readability)
 */
const formattedClassName = computed(() => {
  return props.className.toLowerCase()
})
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
              1d{{ hitDie }} per {{ formattedClassName }} level
            </dd>
          </div>

          <div class="flex flex-col sm:flex-row sm:gap-2">
            <dt class="font-medium text-gray-600 dark:text-gray-400 sm:w-32">
              HP at 1st Level
            </dt>
            <dd class="text-gray-900 dark:text-gray-100">
              {{ hitDie }} + Constitution modifier
            </dd>
          </div>

          <div class="flex flex-col sm:flex-row sm:gap-2">
            <dt class="font-medium text-gray-600 dark:text-gray-400 sm:w-32">
              HP at Higher Levels
            </dt>
            <dd class="text-gray-900 dark:text-gray-100">
              1d{{ hitDie }} (or {{ averageHp }}) + Constitution modifier per {{ formattedClassName }} level after 1st
            </dd>
          </div>
        </dl>
      </div>
    </div>
  </UCard>
</template>
```

**Step 2: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/ui/class/UiHitPointsCard.test.ts`

Expected: All tests PASS

**Step 3: Commit implementation**

```bash
git add app/components/ui/class/UiHitPointsCard.vue
git commit -m "feat: Add UiHitPointsCard component (GREEN)"
```

---

## Task 5: UiSubclassCards Component - Test Setup

**Files:**
- Create: `tests/components/ui/class/UiSubclassCards.test.ts`

**Step 1: Write the failing test file**

```typescript
import { describe, it, expect } from 'vitest'
import { mount, RouterLinkStub } from '@vue/test-utils'
import UiSubclassCards from '~/components/ui/class/UiSubclassCards.vue'

describe('UiSubclassCards', () => {
  const mountOptions = {
    global: {
      stubs: {
        UCard: {
          template: '<div class="card"><slot /><slot name="footer" /></div>'
        },
        UBadge: {
          template: '<span class="badge"><slot /></span>',
          props: ['color', 'variant', 'size']
        },
        UIcon: {
          template: '<i class="icon" />',
          props: ['name']
        },
        NuxtLink: RouterLinkStub
      }
    }
  }

  const mockSubclasses = [
    {
      id: 86,
      slug: 'rogue-arcane-trickster',
      name: 'Arcane Trickster',
      description: 'Subclass of Rogue',
      features: [
        { id: 1, level: 3, feature_name: 'Spellcasting' },
        { id: 2, level: 9, feature_name: 'Magical Ambush' },
        { id: 3, level: 13, feature_name: 'Versatile Trickster' },
        { id: 4, level: 17, feature_name: 'Spell Thief' }
      ],
      sources: [{ id: 1, name: "Player's Handbook", abbreviation: 'PHB', page_number: 97 }]
    },
    {
      id: 87,
      slug: 'rogue-assassin',
      name: 'Assassin',
      description: 'Subclass of Rogue',
      features: [
        { id: 5, level: 3, feature_name: 'Assassinate' },
        { id: 6, level: 9, feature_name: 'Infiltration Expertise' }
      ],
      sources: [{ id: 1, name: "Player's Handbook", abbreviation: 'PHB', page_number: 97 }]
    }
  ]

  it('renders subclass names', () => {
    const wrapper = mount(UiSubclassCards, {
      props: {
        subclasses: mockSubclasses,
        basePath: '/classes'
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('Arcane Trickster')
    expect(wrapper.text()).toContain('Assassin')
  })

  it('renders correct number of cards', () => {
    const wrapper = mount(UiSubclassCards, {
      props: {
        subclasses: mockSubclasses,
        basePath: '/classes'
      },
      ...mountOptions
    })

    const cards = wrapper.findAll('.card')
    expect(cards.length).toBe(2)
  })

  it('links to subclass detail page', () => {
    const wrapper = mount(UiSubclassCards, {
      props: {
        subclasses: mockSubclasses,
        basePath: '/classes'
      },
      ...mountOptions
    })

    const links = wrapper.findAllComponents(RouterLinkStub)
    expect(links[0].props('to')).toBe('/classes/rogue-arcane-trickster')
    expect(links[1].props('to')).toBe('/classes/rogue-assassin')
  })

  it('shows feature count', () => {
    const wrapper = mount(UiSubclassCards, {
      props: {
        subclasses: mockSubclasses,
        basePath: '/classes'
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('4 features')
    expect(wrapper.text()).toContain('2 features')
  })

  it('shows source abbreviation when available', () => {
    const wrapper = mount(UiSubclassCards, {
      props: {
        subclasses: mockSubclasses,
        basePath: '/classes'
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('PHB')
  })

  it('handles subclasses without sources gracefully', () => {
    const noSourceSubclasses = [
      {
        id: 88,
        slug: 'rogue-thief',
        name: 'Thief',
        description: 'Subclass of Rogue',
        features: [],
        sources: []
      }
    ]

    const wrapper = mount(UiSubclassCards, {
      props: {
        subclasses: noSourceSubclasses,
        basePath: '/classes'
      },
      ...mountOptions
    })

    expect(wrapper.text()).toContain('Thief')
    // Should not throw error
  })

  it('uses grid layout for cards', () => {
    const wrapper = mount(UiSubclassCards, {
      props: {
        subclasses: mockSubclasses,
        basePath: '/classes'
      },
      ...mountOptions
    })

    const grid = wrapper.find('.grid')
    expect(grid.exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/ui/class/UiSubclassCards.test.ts`

Expected: FAIL (component doesn't exist)

**Step 3: Commit test file**

```bash
git add tests/components/ui/class/UiSubclassCards.test.ts
git commit -m "test: Add UiSubclassCards test scaffolding (RED)"
```

---

## Task 6: UiSubclassCards Component - Implementation

**Files:**
- Create: `app/components/ui/class/UiSubclassCards.vue`

**Step 1: Write minimal implementation**

```vue
<script setup lang="ts">
interface Feature {
  id: number
  level: number
  feature_name: string
}

interface Source {
  id: number
  name: string
  abbreviation?: string
  page_number?: number
}

interface Subclass {
  id: number | string
  slug: string
  name: string
  description?: string
  features?: Feature[]
  sources?: Source[]
}

interface Props {
  subclasses: Subclass[]
  basePath: string
}

defineProps<Props>()

/**
 * Get source abbreviation for display
 */
const getSourceAbbreviation = (subclass: Subclass): string | null => {
  if (!subclass.sources || subclass.sources.length === 0) return null
  const source = subclass.sources[0]
  return source.abbreviation || null
}

/**
 * Get feature count text
 */
const getFeatureCountText = (subclass: Subclass): string => {
  const count = subclass.features?.length || 0
  return count === 1 ? '1 feature' : `${count} features`
}
</script>

<template>
  <div class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4">
    <NuxtLink
      v-for="subclass in subclasses"
      :key="subclass.id"
      :to="`${basePath}/${subclass.slug}`"
      class="group"
    >
      <UCard
        class="h-full transition-all duration-200 group-hover:ring-2 group-hover:ring-primary-500 group-hover:shadow-lg"
      >
        <div class="space-y-3">
          <!-- Subclass Name -->
          <h4 class="font-semibold text-gray-900 dark:text-gray-100 group-hover:text-primary-600 dark:group-hover:text-primary-400 transition-colors">
            {{ subclass.name }}
          </h4>

          <!-- Meta Info -->
          <div class="flex items-center gap-2 flex-wrap text-sm">
            <UBadge
              v-if="getSourceAbbreviation(subclass)"
              color="neutral"
              variant="subtle"
              size="xs"
            >
              {{ getSourceAbbreviation(subclass) }}
            </UBadge>

            <span class="text-gray-500 dark:text-gray-400">
              {{ getFeatureCountText(subclass) }}
            </span>
          </div>
        </div>

        <template #footer>
          <div class="flex items-center justify-end text-sm text-primary-600 dark:text-primary-400">
            <span class="group-hover:underline">View Details</span>
            <UIcon
              name="i-heroicons-arrow-right"
              class="w-4 h-4 ml-1 transition-transform group-hover:translate-x-1"
            />
          </div>
        </template>
      </UCard>
    </NuxtLink>
  </div>
</template>
```

**Step 2: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/ui/class/UiSubclassCards.test.ts`

Expected: All tests PASS

**Step 3: Commit implementation**

```bash
git add app/components/ui/class/UiSubclassCards.vue
git commit -m "feat: Add UiSubclassCards component (GREEN)"
```

---

## Task 7: Integrate Components into Classes Detail Page

**Files:**
- Modify: `app/pages/classes/[slug].vue`

**Step 1: Read current implementation and understand structure**

The current page has:
- Header with badges
- Quick Stats + Image grid
- Description from first trait
- UAccordion with multiple sections

**Step 2: Update the page with new components**

Replace the content of `app/pages/classes/[slug].vue` with:

```vue
<script setup lang="ts">
import type { CharacterClass } from '~/types/api/entities'
import type { BadgeColor, BadgeSize, BadgeVariant } from '~/utils/badgeColors'

const route = useRoute()

// Fetch class data and setup SEO
const { data: entity, loading, error } = useEntityDetail<CharacterClass>({
  slug: route.params.slug as string,
  endpoint: '/classes',
  cacheKey: 'class',
  seo: {
    titleTemplate: name => `${name} - D&D 5e Class`,
    descriptionExtractor: (charClass: unknown) => {
      const c = charClass as { description?: string }
      return c.description?.substring(0, 160) || ''
    },
    fallbackTitle: 'Class - D&D 5e Compendium'
  }
})

/**
 * Get entity image path (512px variant)
 */
const { getImagePath } = useEntityImage()
const imagePath = computed(() => {
  if (!entity.value) return null
  return getImagePath('classes', entity.value.slug, 512)
})

/**
 * Filter base class features (not subclass features)
 * These are features where is_optional is false or undefined
 */
const baseClassFeatures = computed(() => {
  if (!entity.value?.features) return []
  return entity.value.features.filter(f => !f.is_optional)
})

/**
 * Accordion items with icons
 */
const accordionItems = computed(() => {
  if (!entity.value) return []

  const items = []

  // Proficiencies
  if (entity.value.proficiencies && entity.value.proficiencies.length > 0) {
    items.push({
      label: `Proficiencies (${entity.value.proficiencies.length})`,
      icon: 'i-heroicons-academic-cap',
      slot: 'proficiencies'
    })
  }

  // Equipment
  if (entity.value.equipment && entity.value.equipment.length > 0) {
    items.push({
      label: 'Starting Equipment',
      icon: 'i-heroicons-briefcase',
      slot: 'equipment'
    })
  }

  // Features (detailed descriptions)
  if (baseClassFeatures.value.length > 0) {
    items.push({
      label: `Features (${baseClassFeatures.value.length})`,
      icon: 'i-heroicons-bolt',
      slot: 'features'
    })
  }

  // Counters (if any beyond what's shown in progression)
  if (entity.value.counters && entity.value.counters.length > 0) {
    items.push({
      label: 'Class Resources',
      icon: 'i-heroicons-calculator',
      slot: 'counters'
    })
  }

  // Spell Slot Progression (for casters)
  if (entity.value.level_progression && entity.value.level_progression.length > 0) {
    items.push({
      label: 'Spell Slot Progression',
      icon: 'i-heroicons-sparkles',
      slot: 'spell-progression'
    })
  }

  // Source
  if (entity.value.sources && entity.value.sources.length > 0) {
    items.push({
      label: 'Source',
      icon: 'i-heroicons-book-open',
      slot: 'source'
    })
  }

  return items
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <UiDetailPageLoading
      v-if="loading"
      entity-type="class"
    />

    <UiDetailPageError
      v-else-if="error"
      entity-type="Class"
    />

    <div
      v-else-if="entity"
      class="space-y-8"
    >
      <!-- Breadcrumb Navigation -->
      <UiBackLink
        to="/classes"
        label="Back to Classes"
      />

      <!-- Header -->
      <UiDetailPageHeader
        :title="entity.name"
        :badges="[
          { label: entity.is_base_class ? 'Base Class' : 'Subclass', color: (entity.is_base_class ? 'success' : 'warning') as BadgeColor, variant: 'subtle' as BadgeVariant, size: 'lg' as BadgeSize },
          ...(entity.spellcasting_ability ? [{ label: `Spellcaster (${entity.spellcasting_ability.code})`, color: 'primary' as BadgeColor, variant: 'soft' as BadgeVariant, size: 'sm' as BadgeSize }] : [])
        ]"
      />

      <!-- Quick Stats (2/3) + Image (1/3) Side-by-Side -->
      <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
        <!-- Quick Stats - 2/3 width on large screens -->
        <div class="lg:col-span-2">
          <UiDetailQuickStatsCard
            :columns="2"
            :stats="[
              ...(entity.hit_die ? [{ icon: 'i-heroicons-heart', label: 'Hit Die', value: `1d${entity.hit_die}` }] : []),
              ...(entity.primary_ability ? [{ icon: 'i-heroicons-star', label: 'Primary Ability', value: entity.primary_ability }] : []),
              ...(entity.spellcasting_ability ? [{ icon: 'i-heroicons-sparkles', label: 'Spellcasting Ability', value: `${entity.spellcasting_ability.name} (${entity.spellcasting_ability.code})` }] : [])
            ]"
          />
        </div>

        <!-- Standalone Image - 1/3 width on large screens -->
        <div class="lg:col-span-1">
          <UiDetailEntityImage
            v-if="imagePath"
            :image-path="imagePath"
            :image-alt="`${entity.name} class illustration`"
          />
        </div>
      </div>

      <!-- Class Progression Table (NEW - prominent position) -->
      <UiClassProgressionTable
        v-if="entity.is_base_class && baseClassFeatures.length > 0"
        :features="baseClassFeatures"
        :counters="entity.counters || []"
      />

      <!-- Description -->
      <UiDetailDescriptionCard
        v-if="entity.description"
        :description="entity.description"
      />

      <!-- Hit Points Card (NEW) -->
      <UiHitPointsCard
        v-if="entity.hit_die && entity.is_base_class"
        :hit-die="entity.hit_die"
        :class-name="entity.name"
      />

      <!-- Subclasses Card Grid (NEW - for base classes) -->
      <div
        v-if="entity.is_base_class && entity.subclasses && entity.subclasses.length > 0"
        class="space-y-4"
      >
        <h3 class="text-lg font-semibold text-gray-900 dark:text-gray-100 flex items-center gap-2">
          <UIcon
            name="i-heroicons-users"
            class="w-5 h-5 text-gray-400"
          />
          Subclasses ({{ entity.subclasses.length }})
        </h3>
        <UiSubclassCards
          :subclasses="entity.subclasses"
          base-path="/classes"
        />
      </div>

      <!-- Additional Details (Accordion with icons) -->
      <UAccordion
        v-if="accordionItems.length > 0"
        :items="accordionItems"
        type="multiple"
      >
        <!-- Proficiencies Slot -->
        <template
          v-if="entity.proficiencies && entity.proficiencies.length > 0"
          #proficiencies
        >
          <UiAccordionBulletList :items="entity.proficiencies" />
        </template>

        <!-- Equipment Slot -->
        <template
          v-if="entity.equipment && entity.equipment.length > 0"
          #equipment
        >
          <UiAccordionEquipmentList
            :equipment="entity.equipment"
            type="class"
          />
        </template>

        <!-- Features Slot (detailed descriptions) -->
        <template
          v-if="baseClassFeatures.length > 0"
          #features
        >
          <UiAccordionTraitsList
            :traits="baseClassFeatures"
            :show-level="true"
          />
        </template>

        <!-- Counters Slot -->
        <template
          v-if="entity.counters && entity.counters.length > 0"
          #counters
        >
          <UiAccordionClassCounters :counters="entity.counters" />
        </template>

        <!-- Spell Progression Slot -->
        <template
          v-if="entity.level_progression && entity.level_progression.length > 0"
          #spell-progression
        >
          <UiAccordionLevelProgression :level-progression="entity.level_progression" />
        </template>

        <!-- Source Slot -->
        <template
          v-if="entity.sources && entity.sources.length > 0"
          #source
        >
          <UiSourceDisplay :sources="entity.sources" />
        </template>
      </UAccordion>

      <!-- Bottom Navigation -->
      <UiDetailPageBottomNav
        to="/classes"
        label="Back to Classes"
      />

      <!-- JSON Debug Panel -->
      <JsonDebugPanel
        :data="entity"
        title="Class Data"
      />
    </div>
  </div>
</template>
```

**Step 3: Verify page loads correctly**

Run: `curl -s "http://localhost:3000/classes/rogue" -o /dev/null -w "HTTP Status: %{http_code}\n"`

Expected: `HTTP Status: 200`

**Step 4: Visual verification in browser**

Open `http://localhost:3000/classes/rogue` and verify:
- [ ] Class progression table displays with levels 1-20
- [ ] Sneak Attack column shows correct dice (1d6 at level 1, 2d6 at level 3, etc.)
- [ ] Hit Points card shows correct values
- [ ] Subclass cards display with links
- [ ] Accordion items have icons
- [ ] Dark mode works correctly

**Step 5: Commit integration**

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Integrate enhanced components into Classes detail page

- Add UiClassProgressionTable showing level-by-level features and counters
- Add UiHitPointsCard showing HP calculations
- Add UiSubclassCards replacing nested accordion with card grid
- Add icons to accordion headers
- Reorganize page structure for better information hierarchy"
```

---

## Task 8: Run Full Test Suite and Fix Any Issues

**Step 1: Run all tests**

Run: `docker compose exec nuxt npm run test`

Expected: All tests pass

**Step 2: Run type checking**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No type errors

**Step 3: Test all class pages load correctly**

Run:
```bash
for class in fighter wizard cleric rogue paladin ranger barbarian bard druid monk sorcerer warlock; do
  echo -n "$class: "
  curl -s "http://localhost:3000/classes/$class" -o /dev/null -w "%{http_code}\n"
done
```

Expected: All return 200

**Step 4: Commit any fixes**

If fixes were needed:
```bash
git add -A
git commit -m "fix: Address test failures and type errors"
```

---

## Task 9: Update CHANGELOG and Final Commit

**Files:**
- Modify: `CHANGELOG.md`

**Step 1: Add changelog entry**

Add to the top of CHANGELOG.md under `### Added`:

```markdown
### Added
- Class progression table showing features and counters by level (2025-11-26)
- Hit Points card with HP calculation display for classes (2025-11-26)
- Subclass card grid replacing nested accordion for better navigation (2025-11-26)
- Icons on accordion headers across class detail page (2025-11-26)
```

**Step 2: Commit changelog**

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG with class detail page enhancements"
```

---

## Summary

| Task | Component | Type | Est. Time |
|------|-----------|------|-----------|
| 1-2 | UiClassProgressionTable | New component | 15 min |
| 3-4 | UiHitPointsCard | New component | 10 min |
| 5-6 | UiSubclassCards | New component | 10 min |
| 7 | Page integration | Modification | 15 min |
| 8 | Testing & fixes | Verification | 10 min |
| 9 | Documentation | Changelog | 5 min |

**Total estimated time:** ~65 minutes

**Files created:**
- `app/components/ui/class/UiClassProgressionTable.vue`
- `app/components/ui/class/UiHitPointsCard.vue`
- `app/components/ui/class/UiSubclassCards.vue`
- `tests/components/ui/class/UiClassProgressionTable.test.ts`
- `tests/components/ui/class/UiHitPointsCard.test.ts`
- `tests/components/ui/class/UiSubclassCards.test.ts`

**Files modified:**
- `app/pages/classes/[slug].vue`
- `CHANGELOG.md`
