# Accordion Table Normalization Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a reusable base table component and refactor 3 existing accordion table components to use it, normalizing styling and adding mobile responsiveness.

**Architecture:** Build UiAccordionDataTable as a generic table renderer with slot-based customization. Refactor UiAccordionClassCounters, UiAccordionLevelProgression, and UiAccordionRandomTablesList to use it while preserving their domain logic. Move 4 legacy components from `/ui/` to `/ui/accordion/` for consistent organization.

**Tech Stack:** Vue 3 Composition API, TypeScript, Vitest, @nuxt/test-utils, Tailwind CSS via NuxtUI

---

## Task 1: Create UiAccordionDataTable Base Component (TDD)

**Files:**
- Create: `tests/components/ui/accordion/UiAccordionDataTable.test.ts`
- Create: `app/components/ui/accordion/UiAccordionDataTable.vue`

### Step 1: Write failing test for basic table rendering

Create test file with basic rendering test:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionDataTable from '~/components/ui/accordion/UiAccordionDataTable.vue'

describe('UiAccordionDataTable', () => {
  const columns = [
    { key: 'name', label: 'Name' },
    { key: 'value', label: 'Value' }
  ]

  const rows = [
    { name: 'Item 1', value: '100' },
    { name: 'Item 2', value: '200' }
  ]

  it('renders table with columns and rows', async () => {
    const wrapper = await mountSuspended(UiAccordionDataTable, {
      props: { columns, rows }
    })

    // Check table exists
    expect(wrapper.find('table').exists()).toBe(true)

    // Check headers
    expect(wrapper.text()).toContain('Name')
    expect(wrapper.text()).toContain('Value')

    // Check data
    expect(wrapper.text()).toContain('Item 1')
    expect(wrapper.text()).toContain('100')
    expect(wrapper.text()).toContain('Item 2')
    expect(wrapper.text()).toContain('200')
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm run test -- UiAccordionDataTable
```

Expected: FAIL - Component doesn't exist

### Step 3: Write minimal implementation

Create component file:

```vue
<script setup lang="ts">
interface Column {
  key: string
  label: string
  align?: 'left' | 'center' | 'right'
  width?: string
  cellClass?: string
}

interface Props {
  columns: Column[]
  rows: Record<string, any>[]
  mobileLayout?: 'stacked' | 'cards'
  striped?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  mobileLayout: 'cards',
  striped: true
})

/**
 * Get alignment class for column
 */
const getAlignClass = (align?: string): string => {
  if (align === 'center') return 'text-center'
  if (align === 'right') return 'text-right'
  return 'text-left'
}
</script>

<template>
  <div>
    <!-- Desktop Table -->
    <div class="hidden md:block overflow-x-auto">
      <table class="min-w-full divide-y divide-gray-200 dark:divide-gray-700">
        <thead class="bg-gray-50 dark:bg-gray-800">
          <tr>
            <th
              v-for="column in columns"
              :key="column.key"
              :class="[
                'px-4 py-3 text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider',
                getAlignClass(column.align),
                column.width
              ]"
            >
              {{ column.label }}
            </th>
          </tr>
        </thead>
        <tbody class="bg-white dark:bg-gray-900 divide-y divide-gray-200 dark:divide-gray-700">
          <tr
            v-for="(row, index) in rows"
            :key="index"
            :class="striped && index % 2 === 1 ? 'bg-gray-50 dark:bg-gray-800/30' : ''"
          >
            <td
              v-for="column in columns"
              :key="column.key"
              :class="[
                'px-4 py-3 whitespace-nowrap text-sm text-gray-900 dark:text-gray-100',
                getAlignClass(column.align),
                column.cellClass
              ]"
            >
              <slot
                :name="`cell-${column.key}`"
                :value="row[column.key]"
                :row="row"
              >
                {{ row[column.key] }}
              </slot>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Mobile Cards -->
    <div class="md:hidden space-y-4">
      <div
        v-for="(row, index) in rows"
        :key="index"
        class="p-4 rounded-lg bg-gray-50 dark:bg-gray-800 space-y-2"
      >
        <div
          v-for="column in columns"
          :key="column.key"
          class="flex justify-between items-center"
        >
          <span class="text-sm font-medium text-gray-500 dark:text-gray-400">
            {{ column.label }}:
          </span>
          <span class="text-sm text-gray-900 dark:text-gray-100">
            <slot
              :name="`cell-${column.key}`"
              :value="row[column.key]"
              :row="row"
            >
              {{ row[column.key] }}
            </slot>
          </span>
        </div>
      </div>
    </div>
  </div>
</template>
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm run test -- UiAccordionDataTable
```

Expected: PASS (1 test)

### Step 5: Add test for custom cell slots

Add to test file:

```typescript
it('renders custom cell content via slots', async () => {
  const wrapper = await mountSuspended(UiAccordionDataTable, {
    props: { columns, rows },
    slots: {
      'cell-value': '<span class="custom-badge">{{ value }}</span>'
    }
  })

  expect(wrapper.find('.custom-badge').exists()).toBe(true)
})
```

### Step 6: Run test to verify slots work

```bash
docker compose exec nuxt npm run test -- UiAccordionDataTable
```

Expected: PASS (2 tests)

### Step 7: Add test for empty state

Add to test file:

```typescript
it('handles empty rows array', async () => {
  const wrapper = await mountSuspended(UiAccordionDataTable, {
    props: { columns, rows: [] }
  })

  expect(wrapper.find('table').exists()).toBe(true)
  expect(wrapper.find('tbody tr').exists()).toBe(false)
})
```

### Step 8: Run test

```bash
docker compose exec nuxt npm run test -- UiAccordionDataTable
```

Expected: PASS (3 tests)

### Step 9: Add test for striped rows

Add to test file:

```typescript
it('applies striped styling to alternating rows', async () => {
  const wrapper = await mountSuspended(UiAccordionDataTable, {
    props: { columns, rows, striped: true }
  })

  const tableRows = wrapper.findAll('tbody tr')
  expect(tableRows[0].classes()).not.toContain('bg-gray-50')
  expect(tableRows[1].classes()).toContain('bg-gray-50')
})
```

### Step 10: Run test

```bash
docker compose exec nuxt npm run test -- UiAccordionDataTable
```

Expected: PASS (4 tests)

### Step 11: Commit base component

```bash
git add app/components/ui/accordion/UiAccordionDataTable.vue tests/components/ui/accordion/UiAccordionDataTable.test.ts
git commit -m "feat: Add UiAccordionDataTable base component with tests

- Generic table renderer for accordion content
- Desktop table + mobile card layout
- Standardized padding (px-4 py-3) and colors
- Slot-based custom cell rendering
- 4 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Refactor UiAccordionClassCounters (TDD)

**Files:**
- Modify: `tests/components/ui/UiAccordionClassCounters.test.ts`
- Modify: `app/components/ui/UiAccordionClassCounters.vue`

### Step 1: Read existing test file

```bash
cat tests/components/ui/UiAccordionClassCounters.test.ts
```

Understand current test coverage.

### Step 2: Update tests to expect new structure

Modify test file to expect the new base component structure while maintaining same props interface:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionClassCounters from '~/components/ui/UiAccordionClassCounters.vue'
import type { ClassCounterResource } from '~/types/api/entities'

describe('UiAccordionClassCounters', () => {
  const mockCounters: ClassCounterResource[] = [
    {
      id: 1,
      level: 3,
      counter_name: 'Rage',
      counter_value: 3,
      reset_timing: 'Long Rest'
    },
    {
      id: 2,
      level: 1,
      counter_name: 'Ki Points',
      counter_value: 2,
      reset_timing: 'Short Rest'
    }
  ]

  it('renders counters sorted by level', async () => {
    const wrapper = await mountSuspended(UiAccordionClassCounters, {
      props: { counters: mockCounters }
    })

    expect(wrapper.text()).toContain('Ki Points')
    expect(wrapper.text()).toContain('Rage')

    // Level 1 should appear before Level 3 (sorted)
    const text = wrapper.text()
    const kiIndex = text.indexOf('Ki Points')
    const rageIndex = text.indexOf('Rage')
    expect(kiIndex).toBeLessThan(rageIndex)
  })

  it('displays reset timing badges', async () => {
    const wrapper = await mountSuspended(UiAccordionClassCounters, {
      props: { counters: mockCounters }
    })

    expect(wrapper.text()).toContain('Long Rest')
    expect(wrapper.text()).toContain('Short Rest')
  })

  it('handles empty counters array', async () => {
    const wrapper = await mountSuspended(UiAccordionClassCounters, {
      props: { counters: [] }
    })

    expect(wrapper.text()).toBe('')
  })
})
```

### Step 3: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- UiAccordionClassCounters
```

Expected: Tests may pass or fail depending on structure - we're establishing baseline

### Step 4: Refactor component to use base table

Replace component implementation:

```vue
<script setup lang="ts">
import type { ClassCounterResource } from '~/types/api/entities'
import { getResetTimingColor } from '~/utils/badgeColors'

interface Props {
  counters: ClassCounterResource[]
}

const props = defineProps<Props>()

/**
 * Sort counters by level ascending
 */
const sortedCounters = computed(() => {
  if (!props.counters) return []
  return [...props.counters].sort((a, b) => a.level - b.level)
})

/**
 * Column definitions for the table
 */
const columns = [
  { key: 'level', label: 'Level', width: 'w-20' },
  { key: 'counter_name', label: 'Counter' },
  { key: 'counter_value', label: 'Value', width: 'w-24' },
  { key: 'reset_timing', label: 'Reset Timing' }
]
</script>

<template>
  <div
    v-if="counters && counters.length > 0"
    class="p-4"
  >
    <UiAccordionDataTable
      :columns="columns"
      :rows="sortedCounters"
    >
      <template #cell-reset_timing="{ value }">
        <UBadge
          :color="getResetTimingColor(value)"
          variant="soft"
          size="sm"
        >
          {{ value }}
        </UBadge>
      </template>
    </UiAccordionDataTable>
  </div>
</template>
```

### Step 5: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- UiAccordionClassCounters
```

Expected: PASS (all tests)

### Step 6: Commit refactored component

```bash
git add app/components/ui/UiAccordionClassCounters.vue tests/components/ui/UiAccordionClassCounters.test.ts
git commit -m "refactor: UiAccordionClassCounters to use base table

- Reduced from 103 to ~50 lines
- Uses UiAccordionDataTable base component
- Preserves sorting logic and badge rendering
- Gains mobile card layout automatically
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Refactor UiAccordionLevelProgression (TDD)

**Files:**
- Modify: `tests/components/ui/accordion/UiAccordionLevelProgression.test.ts`
- Modify: `app/components/ui/accordion/UiAccordionLevelProgression.vue`

### Step 1: Check if test file exists

```bash
ls tests/components/ui/accordion/UiAccordionLevelProgression.test.ts
```

If not exists, we'll create it in next step.

### Step 2: Create or update test file

Create/update test file:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionLevelProgression from '~/components/ui/accordion/UiAccordionLevelProgression.vue'
import type { components } from '~/types/api/generated'

type LevelProgression = components['schemas']['ClassLevelProgressionResource']

describe('UiAccordionLevelProgression', () => {
  const mockProgression: LevelProgression[] = [
    {
      id: 1,
      level: 1,
      cantrips_known: 3,
      spells_known: 2,
      spell_slots_1st: 2,
      spell_slots_2nd: 0,
      spell_slots_3rd: 0,
      spell_slots_4th: 0,
      spell_slots_5th: 0,
      spell_slots_6th: 0,
      spell_slots_7th: 0,
      spell_slots_8th: 0,
      spell_slots_9th: 0
    },
    {
      id: 2,
      level: 2,
      cantrips_known: 3,
      spells_known: 3,
      spell_slots_1st: 3,
      spell_slots_2nd: 0,
      spell_slots_3rd: 0,
      spell_slots_4th: 0,
      spell_slots_5th: 0,
      spell_slots_6th: 0,
      spell_slots_7th: 0,
      spell_slots_8th: 0,
      spell_slots_9th: 0
    }
  ]

  it('renders level progression table', async () => {
    const wrapper = await mountSuspended(UiAccordionLevelProgression, {
      props: { levelProgression: mockProgression }
    })

    expect(wrapper.text()).toContain('Level')
    expect(wrapper.text()).toContain('Cantrips')
    expect(wrapper.text()).toContain('1st')
  })

  it('displays spell slot values', async () => {
    const wrapper = await mountSuspended(UiAccordionLevelProgression, {
      props: { levelProgression: mockProgression }
    })

    expect(wrapper.text()).toContain('2')
    expect(wrapper.text()).toContain('3')
  })

  it('hides columns with all zero values', async () => {
    const wrapper = await mountSuspended(UiAccordionLevelProgression, {
      props: { levelProgression: mockProgression }
    })

    // Should show 1st level slots
    expect(wrapper.text()).toContain('1st')

    // Should NOT show 9th level slots (all zeros)
    expect(wrapper.text()).not.toContain('9th')
  })

  it('handles empty progression array', async () => {
    const wrapper = await mountSuspended(UiAccordionLevelProgression, {
      props: { levelProgression: [] }
    })

    expect(wrapper.text()).toBe('')
  })
})
```

### Step 3: Run tests to establish baseline

```bash
docker compose exec nuxt npm run test -- UiAccordionLevelProgression
```

Expected: May pass or fail - establishing baseline

### Step 4: Refactor component to use base table

Replace component implementation:

```vue
<script setup lang="ts">
import type { components } from '~/types/api/generated'

type LevelProgression = components['schemas']['ClassLevelProgressionResource']

interface Props {
  levelProgression: LevelProgression[]
  borderColor?: string
}

const props = withDefaults(defineProps<Props>(), {
  borderColor: 'gray-700'
})

// Don't render if no data
if (!props.levelProgression || props.levelProgression.length === 0) {
  // Component returns nothing
}

// Determine which columns to show based on data
const hasCantrips = computed(() =>
  props.levelProgression.some(level => level.cantrips_known !== null && level.cantrips_known !== 0)
)

const hasSpellsKnown = computed(() =>
  props.levelProgression.some(level => 'spells_known' in level)
)

// Check which spell level columns should be shown (hide if all 0)
const showSpellLevel = (level: number) => {
  const key = `spell_slots_${level === 1 ? '1st' : level === 2 ? '2nd' : level === 3 ? '3rd' : `${level}th`}` as keyof LevelProgression
  return props.levelProgression.some((prog) => {
    const value = prog[key]
    return value !== null && value !== 0
  })
}

const visibleSpellLevels = computed(() => {
  const levels = []
  for (let i = 1; i <= 9; i++) {
    if (showSpellLevel(i)) {
      levels.push(i)
    }
  }
  return levels
})

// Helper to format spell level ordinals
const ordinalSuffix = (n: number): string => {
  if (n === 1) return '1st'
  if (n === 2) return '2nd'
  if (n === 3) return '3rd'
  return `${n}th`
}

// Helper to display null values as em dash
const displayValue = (value: number | null): string => {
  if (value === null) return '—'
  return String(value)
}

// Get spell slot value for a given level
const getSpellSlot = (progression: LevelProgression, level: number): number | null => {
  const key = `spell_slots_${ordinalSuffix(level)}` as keyof LevelProgression
  return progression[key] as number | null
}

// Build dynamic columns array
const columns = computed(() => {
  const cols = [
    { key: 'level', label: 'Level', width: 'w-20' }
  ]

  if (hasCantrips.value) {
    cols.push({ key: 'cantrips_known', label: 'Cantrips' })
  }

  if (hasSpellsKnown.value) {
    cols.push({ key: 'spells_known', label: 'Spells Known' })
  }

  for (const spellLevel of visibleSpellLevels.value) {
    cols.push({
      key: `spell_level_${spellLevel}`,
      label: ordinalSuffix(spellLevel),
      align: 'center' as const
    })
  }

  return cols
})

// Transform progression data to match dynamic columns
const tableRows = computed(() => {
  return props.levelProgression.map(prog => {
    const row: Record<string, any> = {
      level: prog.level,
      cantrips_known: displayValue(prog.cantrips_known),
      spells_known: displayValue(prog.spells_known)
    }

    // Add spell slot columns
    for (const spellLevel of visibleSpellLevels.value) {
      row[`spell_level_${spellLevel}`] = getSpellSlot(prog, spellLevel)
    }

    return row
  })
})
</script>

<template>
  <div
    v-if="levelProgression && levelProgression.length > 0"
    class="overflow-x-auto"
  >
    <UiAccordionDataTable
      :columns="columns"
      :rows="tableRows"
    />
  </div>
</template>
```

### Step 5: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- UiAccordionLevelProgression
```

Expected: PASS (all tests)

### Step 6: Commit refactored component

```bash
git add app/components/ui/accordion/UiAccordionLevelProgression.vue tests/components/ui/accordion/UiAccordionLevelProgression.test.ts
git commit -m "refactor: UiAccordionLevelProgression to use base table

- Reduced from 143 to ~80 lines
- Uses UiAccordionDataTable base component
- Preserves dynamic column logic
- Gains mobile card layout automatically
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Refactor UiAccordionRandomTablesList (TDD)

**Files:**
- Modify: `tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts`
- Modify: `app/components/ui/accordion/UiAccordionRandomTablesList.vue`

### Step 1: Check if test file exists

```bash
ls tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts
```

If not exists, create in next step.

### Step 2: Create or update test file

Create/update test file:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionRandomTablesList from '~/components/ui/accordion/UiAccordionRandomTablesList.vue'
import type { components } from '~/types/api/generated'

type RandomTableResource = components['schemas']['RandomTableResource']

describe('UiAccordionRandomTablesList', () => {
  const mockTables: RandomTableResource[] = [
    {
      id: 1,
      table_name: 'Treasure Type',
      dice_type: 'd6',
      description: 'Roll to determine treasure',
      entries: [
        { id: 1, roll_min: 1, roll_max: 2, result_text: 'Gold' },
        { id: 2, roll_min: 3, roll_max: 4, result_text: 'Silver' },
        { id: 3, roll_min: 5, roll_max: 6, result_text: 'Copper' }
      ]
    }
  ]

  it('renders table name and dice type', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    expect(wrapper.text()).toContain('Treasure Type')
    expect(wrapper.text()).toContain('d6')
  })

  it('displays roll ranges and results', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockTables }
    })

    expect(wrapper.text()).toContain('1-2')
    expect(wrapper.text()).toContain('Gold')
    expect(wrapper.text()).toContain('5-6')
    expect(wrapper.text()).toContain('Copper')
  })

  it('handles pipe-delimited columns', async () => {
    const mockWithPipes: RandomTableResource[] = [
      {
        id: 1,
        table_name: 'Multi-column',
        dice_type: 'd4',
        entries: [
          { id: 1, roll_min: 1, roll_max: 1, result_text: 'House Cannith | Alchemist supplies' },
          { id: 2, roll_min: 2, roll_max: 2, result_text: 'House Deneith | Smith tools' }
        ]
      }
    ]

    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: mockWithPipes }
    })

    expect(wrapper.text()).toContain('House Cannith')
    expect(wrapper.text()).toContain('Alchemist supplies')
  })

  it('handles empty tables array', async () => {
    const wrapper = await mountSuspended(UiAccordionRandomTablesList, {
      props: { tables: [] }
    })

    expect(wrapper.text()).toBe('')
  })
})
```

### Step 3: Run tests to establish baseline

```bash
docker compose exec nuxt npm run test -- UiAccordionRandomTablesList
```

Expected: May pass or fail - establishing baseline

### Step 4: Refactor component to use base table

Replace component implementation:

```vue
<script setup lang="ts">
import type { components } from '~/types/api/generated'

type RandomTableResource = components['schemas']['RandomTableResource']

interface Props {
  tables: RandomTableResource[]
  borderColor?: string
}

withDefaults(defineProps<Props>(), {
  borderColor: 'primary-500'
})

/**
 * Format roll range display (e.g., "5" or "3-6")
 */
const formatRollRange = (min: number | null, max: number | null): string => {
  if (min === null || max === null) return ''
  return min === max ? `${min}` : `${min}-${max}`
}

/**
 * Check if a table has any dice rolls (non-null roll_min/roll_max)
 */
const hasRolls = (table: RandomTableResource): boolean => {
  if (!table.entries || table.entries.length === 0) return false
  return table.entries.some(entry => entry.roll_min !== null || entry.roll_max !== null)
}

/**
 * Parse pipe-delimited result_text into columns
 */
const parseColumns = (resultText: string | null): string[] => {
  if (!resultText) return ['']
  return resultText.split('|').map(col => col.trim())
}

/**
 * Get column count for a table (max number of pipe-separated columns)
 */
const getColumnCount = (table: RandomTableResource): number => {
  if (!table.entries || table.entries.length === 0) return 1
  const maxColumns = Math.max(...table.entries.map(entry => parseColumns(entry.result_text).length))
  return maxColumns
}

/**
 * Build columns for a specific table
 */
const buildColumns = (table: RandomTableResource) => {
  const cols = []

  if (hasRolls(table)) {
    cols.push({ key: 'roll', label: 'Roll', width: 'w-24' })
  }

  const columnCount = getColumnCount(table)
  for (let i = 0; i < columnCount; i++) {
    cols.push({
      key: `col_${i}`,
      label: i === 0 && columnCount === 1 ? 'Result' : ''
    })
  }

  return cols
}

/**
 * Transform table entries to row format
 */
const buildRows = (table: RandomTableResource) => {
  if (!table.entries) return []

  return table.entries.map(entry => {
    const row: Record<string, any> = {}

    if (hasRolls(table)) {
      row.roll = formatRollRange(entry.roll_min, entry.roll_max)
    }

    const columns = parseColumns(entry.result_text)
    columns.forEach((col, index) => {
      row[`col_${index}`] = col
    })

    return row
  })
}
</script>

<template>
  <div
    v-if="tables.length > 0"
    class="space-y-6 p-4"
  >
    <div
      v-for="table in tables"
      :key="table.id"
      class="space-y-2 border-l-4 pl-4"
      :class="`border-${borderColor}`"
    >
      <!-- Table Name and Dice Type -->
      <h4 class="text-base font-semibold text-gray-900 dark:text-gray-100">
        {{ table.table_name }}
        <span
          v-if="table.dice_type"
          class="text-sm font-normal text-gray-600 dark:text-gray-400"
        >
          ({{ table.dice_type }})
        </span>
      </h4>

      <!-- Optional Description -->
      <p
        v-if="table.description"
        class="text-sm text-gray-700 dark:text-gray-300"
      >
        {{ table.description }}
      </p>

      <!-- Table Data -->
      <UiAccordionDataTable
        :columns="buildColumns(table)"
        :rows="buildRows(table)"
        :striped="false"
      />
    </div>
  </div>
</template>
```

### Step 5: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- UiAccordionRandomTablesList
```

Expected: PASS (all tests)

### Step 6: Commit refactored component

```bash
git add app/components/ui/accordion/UiAccordionRandomTablesList.vue tests/components/ui/accordion/UiAccordionRandomTablesList.test.ts
git commit -m "refactor: UiAccordionRandomTablesList to use base table

- Reduced from 134 to ~70 lines
- Uses UiAccordionDataTable base component
- Removed pl-4 container indent (inconsistent)
- Preserves pipe-parsing and roll range logic
- Gains mobile card layout automatically
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Move Legacy Components to /ui/accordion/

**Files:**
- Move: `app/components/ui/UiAccordionClassCounters.vue` → `app/components/ui/accordion/UiAccordionClassCounters.vue`
- Move: `app/components/ui/UiAccordionConditions.vue` → `app/components/ui/accordion/UiAccordionConditions.vue`
- Move: `app/components/ui/UiAccordionItemDetail.vue` → `app/components/ui/accordion/UiAccordionItemDetail.vue`
- Move: `app/components/ui/UiAccordionPrerequisites.vue` → `app/components/ui/accordion/UiAccordionPrerequisites.vue`

### Step 1: Move UiAccordionClassCounters (already done in Task 2)

This was already moved as part of the refactoring in Task 2. Verify:

```bash
ls app/components/ui/accordion/UiAccordionClassCounters.vue
```

Expected: File exists

### Step 2: Move UiAccordionConditions

```bash
git mv app/components/ui/UiAccordionConditions.vue app/components/ui/accordion/UiAccordionConditions.vue
```

### Step 3: Move corresponding test file if it exists

```bash
if [ -f tests/components/ui/UiAccordionConditions.test.ts ]; then
  git mv tests/components/ui/UiAccordionConditions.test.ts tests/components/ui/accordion/UiAccordionConditions.test.ts
fi
```

### Step 4: Move UiAccordionItemDetail

```bash
git mv app/components/ui/UiAccordionItemDetail.vue app/components/ui/accordion/UiAccordionItemDetail.vue
```

### Step 5: Move corresponding test file if it exists

```bash
if [ -f tests/components/ui/UiAccordionItemDetail.test.ts ]; then
  git mv tests/components/ui/UiAccordionItemDetail.test.ts tests/components/ui/accordion/UiAccordionItemDetail.test.ts
fi
```

### Step 6: Move UiAccordionPrerequisites

```bash
git mv app/components/ui/UiAccordionPrerequisites.vue app/components/ui/accordion/UiAccordionPrerequisites.vue
```

### Step 7: Move corresponding test file if it exists

```bash
if [ -f tests/components/ui/UiAccordionPrerequisites.test.ts ]; then
  git mv tests/components/ui/UiAccordionPrerequisites.test.ts tests/components/ui/accordion/UiAccordionPrerequisites.test.ts
fi
```

### Step 8: Run full test suite to verify auto-import still works

```bash
docker compose exec nuxt npm run test
```

Expected: All tests pass - Nuxt auto-import handles the moved components

### Step 9: Commit file organization changes

```bash
git commit -m "refactor: Move 4 legacy accordion components to /ui/accordion/

- Move UiAccordionConditions
- Move UiAccordionItemDetail
- Move UiAccordionPrerequisites
- All 19 accordion components now in consistent location
- Auto-import still works (no breaking changes)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Verification & Documentation

**Files:**
- Modify: `CHANGELOG.md`

### Step 1: Run full test suite

```bash
docker compose exec nuxt npm run test
```

Expected: All tests pass

### Step 2: Run TypeScript type checking

```bash
docker compose exec nuxt npm run typecheck
```

Expected: No errors

### Step 3: Run linter

```bash
docker compose exec nuxt npm run lint
```

Expected: No errors (or run `npm run lint:fix` if minor issues)

### Step 4: Manual browser testing - Classes page

Open browser to http://localhost:3000/classes/1 (or any class detail page)

Verify:
- Class counters table displays correctly
- Level progression table displays correctly
- Mobile responsive layout works (resize browser)
- Dark mode works (toggle theme)

### Step 5: Manual browser testing - Backgrounds page

Open browser to http://localhost:3000/backgrounds/1 (or any background detail page)

Verify:
- Random tables display correctly if present
- Mobile responsive layout works

### Step 6: Manual browser testing - Mobile view

Use browser dev tools to test mobile viewport (375px width)

Verify:
- Tables switch to card layout
- Cards are readable and well-formatted
- All data visible without horizontal scroll

### Step 7: Update CHANGELOG.md

Add entry to CHANGELOG.md:

```markdown
### Changed
- Normalized accordion table components with consistent styling (2025-11-23)
  - Created reusable UiAccordionDataTable base component
  - Refactored UiAccordionClassCounters, UiAccordionLevelProgression, UiAccordionRandomTablesList
  - Standardized padding (px-4 py-3) and colors (bg-gray-50 dark:bg-gray-800)
  - All table components now have mobile-responsive card layouts
  - Reduced table markup code by 47% (377→200 lines)
  - Moved 4 legacy components to /ui/accordion/ for consistency
```

### Step 8: Commit CHANGELOG update

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG for accordion table normalization

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

### Step 9: Final verification

```bash
git log --oneline -10
```

Expected: See all commits from this plan

---

## Success Metrics Verification

After completing all tasks, verify:

✅ Single consistent padding: `px-4 py-3`
✅ Single header color: `bg-gray-50 dark:bg-gray-800`
✅ All 3 tables have mobile support
✅ ~200 lines of table markup code (47% reduction)
✅ All 19 accordion components in `/ui/accordion/`
✅ Reusable base component for future table needs

---

## Rollback Plan

If issues arise, use these commits to rollback specific changes:

```bash
# Rollback everything
git revert HEAD~7..HEAD

# Or rollback specific components
git revert <commit-hash-for-specific-component>
```
