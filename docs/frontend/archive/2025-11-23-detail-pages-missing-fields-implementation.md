# Detail Pages Missing Fields - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add missing API fields to all 7 detail pages using reusable components following TDD principles.

**Architecture:** Component-first approach - create 4 new reusable UI components with comprehensive tests, then integrate them into 5 detail pages. Standardize image display components across all pages.

**Tech Stack:** Vue 3 (Composition API), TypeScript, Vitest, @nuxt/test-utils, NuxtUI 4.x

---

## Overview

**Components to Create:**
1. `UiAccordionConditions` - Display conditions for Races, Feats, Monsters
2. `UiAccordionClassCounters` - Display class resource counters (Rage, Ki Points, etc.)
3. `UiAccordionItemDetail` - Display item detail flavor text
4. `UiAccordionPrerequisites` - Display item prerequisites

**Pages to Update:**
1. `classes/[slug].vue` - Add counters
2. `feats/[slug].vue` - Add proficiencies & conditions
3. `monsters/[slug].vue` - Add conditions
4. `items/[slug].vue` - Add detail & prerequisites
5. `backgrounds/[slug].vue` - Standardize image component

**Color Utilities to Add:**
1. `getResetTimingColor()` - For counter reset timing badges
2. `getConditionEffectColor()` - For condition effect type badges

---

## Task 1: Add Color Utility Functions

**Files:**
- Modify: `app/utils/badgeColors.ts`
- Test: `tests/utils/badgeColors.test.ts` (if exists, otherwise skip test for utilities)

### Step 1: Add reset timing color function

**In `app/utils/badgeColors.ts`**, add after existing functions:

```typescript
/**
 * Get color for class counter reset timing
 */
export function getResetTimingColor(timing: string): BadgeColor {
  switch (timing) {
    case 'Short Rest':
      return 'info'
    case 'Long Rest':
      return 'primary'
    case 'Does Not Reset':
      return 'neutral'
    default:
      return 'neutral'
  }
}

/**
 * Get color for condition effect type
 */
export function getConditionEffectColor(effectType: string): BadgeColor {
  switch (effectType) {
    case 'immunity':
      return 'success'
    case 'resistance':
      return 'info'
    case 'vulnerability':
      return 'error'
    case 'inflicts':
      return 'warning'
    default:
      return 'neutral'
  }
}
```

### Step 2: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 3: Commit

```bash
git add app/utils/badgeColors.ts
git commit -m "feat: Add color utilities for conditions and class counters

- Add getResetTimingColor() for counter reset timing badges
- Add getConditionEffectColor() for condition effect badges

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: UiAccordionConditions Component

**Files:**
- Create: `app/components/ui/UiAccordionConditions.vue`
- Create: `tests/components/ui/UiAccordionConditions.test.ts`

### Step 1: Write the failing test

**Create `tests/components/ui/UiAccordionConditions.test.ts`:**

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionConditions from '~/components/ui/UiAccordionConditions.vue'
import type { EntityConditionResource } from '~/types/api/entities'

describe('UiAccordionConditions', () => {
  const mockConditions: EntityConditionResource[] = [
    {
      id: 1,
      condition_id: 1,
      condition: {
        id: 1,
        name: 'Poisoned',
        slug: 'poisoned',
        description: 'A poisoned creature has disadvantage on attack rolls and ability checks.'
      },
      effect_type: 'immunity',
      description: null
    },
    {
      id: 2,
      condition_id: 2,
      condition: {
        id: 2,
        name: 'Frightened',
        slug: 'frightened',
        description: 'A frightened creature has disadvantage on ability checks and attack rolls.'
      },
      effect_type: 'resistance',
      description: 'While in dim light or darkness'
    }
  ]

  it('renders multiple conditions', async () => {
    const wrapper = await mountSuspended(UiAccordionConditions, {
      props: { conditions: mockConditions }
    })

    expect(wrapper.text()).toContain('Poisoned')
    expect(wrapper.text()).toContain('Frightened')
  })

  it('displays effect type badges', async () => {
    const wrapper = await mountSuspended(UiAccordionConditions, {
      props: { conditions: mockConditions }
    })

    expect(wrapper.text()).toContain('Immunity')
    expect(wrapper.text()).toContain('Resistance')
  })

  it('shows condition descriptions', async () => {
    const wrapper = await mountSuspended(UiAccordionConditions, {
      props: { conditions: mockConditions }
    })

    expect(wrapper.text()).toContain('A poisoned creature has disadvantage')
    expect(wrapper.text()).toContain('A frightened creature has disadvantage')
  })

  it('shows entity-specific description when provided', async () => {
    const wrapper = await mountSuspended(UiAccordionConditions, {
      props: { conditions: mockConditions }
    })

    expect(wrapper.text()).toContain('While in dim light or darkness')
  })

  it('handles empty conditions array', async () => {
    const wrapper = await mountSuspended(UiAccordionConditions, {
      props: { conditions: [] }
    })

    expect(wrapper.text()).toBe('')
  })

  it('handles null/undefined gracefully', async () => {
    const conditionsWithMissing: EntityConditionResource[] = [
      {
        id: 3,
        condition_id: null,
        condition: undefined,
        effect_type: 'immunity',
        description: 'Custom immunity'
      }
    ]

    const wrapper = await mountSuspended(UiAccordionConditions, {
      props: { conditions: conditionsWithMissing }
    })

    expect(wrapper.text()).toContain('Custom immunity')
  })
})
```

### Step 2: Run test to verify it fails

Run: `docker compose exec nuxt npm run test -- UiAccordionConditions`
Expected: FAIL - Component does not exist

### Step 3: Write minimal implementation

**Create `app/components/ui/UiAccordionConditions.vue`:**

```vue
<script setup lang="ts">
import type { EntityConditionResource } from '~/types/api/entities'
import { getConditionEffectColor } from '~/utils/badgeColors'

interface Props {
  conditions: EntityConditionResource[]
  entityType?: 'race' | 'feat' | 'monster'
}

const props = defineProps<Props>()

/**
 * Format effect type for display
 */
const formatEffectType = (effectType: string): string => {
  return effectType.charAt(0).toUpperCase() + effectType.slice(1)
}
</script>

<template>
  <div
    v-if="conditions && conditions.length > 0"
    class="p-4 space-y-3"
  >
    <div
      v-for="conditionRelation in conditions"
      :key="conditionRelation.id"
      class="p-3 rounded-lg bg-gray-50 dark:bg-gray-800"
    >
      <div class="flex items-start gap-3">
        <!-- Condition Name Badge -->
        <UBadge
          :color="getConditionEffectColor(conditionRelation.effect_type)"
          variant="soft"
        >
          {{ conditionRelation.condition?.name || 'Unknown' }}
        </UBadge>

        <!-- Content -->
        <div class="flex-1">
          <!-- Effect Type -->
          <div
            v-if="conditionRelation.effect_type"
            class="text-sm font-medium text-gray-900 dark:text-gray-100 mb-1"
          >
            Effect: {{ formatEffectType(conditionRelation.effect_type) }}
          </div>

          <!-- Condition Description -->
          <div
            v-if="conditionRelation.condition?.description"
            class="text-sm text-gray-700 dark:text-gray-300"
          >
            {{ conditionRelation.condition.description }}
          </div>

          <!-- Entity-Specific Description -->
          <div
            v-if="conditionRelation.description"
            class="text-sm text-gray-600 dark:text-gray-400 mt-2 italic"
          >
            {{ conditionRelation.description }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
```

### Step 4: Run test to verify it passes

Run: `docker compose exec nuxt npm run test -- UiAccordionConditions`
Expected: All tests PASS

### Step 5: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 6: Commit

```bash
git add app/components/ui/UiAccordionConditions.vue tests/components/ui/UiAccordionConditions.test.ts
git commit -m "feat: Add UiAccordionConditions component with tests

- Display conditions with effect type badges
- Show condition descriptions and entity-specific notes
- Handle empty/null states gracefully
- 6 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: UiAccordionClassCounters Component

**Files:**
- Create: `app/components/ui/UiAccordionClassCounters.vue`
- Create: `tests/components/ui/UiAccordionClassCounters.test.ts`

### Step 1: Write the failing test

**Create `tests/components/ui/UiAccordionClassCounters.test.ts`:**

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionClassCounters from '~/components/ui/UiAccordionClassCounters.vue'
import type { ClassCounterResource } from '~/types/api/entities'

describe('UiAccordionClassCounters', () => {
  const mockCounters: ClassCounterResource[] = [
    {
      id: 1,
      level: 1,
      counter_name: 'Rage',
      counter_value: 2,
      reset_timing: 'Long Rest'
    },
    {
      id: 2,
      level: 3,
      counter_name: 'Rage',
      counter_value: 3,
      reset_timing: 'Long Rest'
    },
    {
      id: 3,
      level: 2,
      counter_name: 'Reckless Attack',
      counter_value: 1,
      reset_timing: 'Does Not Reset'
    }
  ]

  it('renders counter table', async () => {
    const wrapper = await mountSuspended(UiAccordionClassCounters, {
      props: { counters: mockCounters }
    })

    expect(wrapper.text()).toContain('Level')
    expect(wrapper.text()).toContain('Counter')
    expect(wrapper.text()).toContain('Value')
    expect(wrapper.text()).toContain('Reset Timing')
  })

  it('displays all counter entries', async () => {
    const wrapper = await mountSuspended(UiAccordionClassCounters, {
      props: { counters: mockCounters }
    })

    expect(wrapper.text()).toContain('Rage')
    expect(wrapper.text()).toContain('Reckless Attack')
    expect(wrapper.text()).toContain('Long Rest')
  })

  it('shows reset timing with badges', async () => {
    const wrapper = await mountSuspended(UiAccordionClassCounters, {
      props: { counters: mockCounters }
    })

    // Find all badge components
    const badges = wrapper.findAllComponents({ name: 'UBadge' })
    expect(badges.length).toBeGreaterThan(0)
  })

  it('sorts counters by level', async () => {
    const wrapper = await mountSuspended(UiAccordionClassCounters, {
      props: { counters: mockCounters }
    })

    const html = wrapper.html()
    // Level 1 should appear before Level 3
    const level1Index = html.indexOf('>1<')
    const level3Index = html.indexOf('>3<')
    expect(level1Index).toBeLessThan(level3Index)
  })

  it('handles empty counters array', async () => {
    const wrapper = await mountSuspended(UiAccordionClassCounters, {
      props: { counters: [] }
    })

    expect(wrapper.text()).toBe('')
  })
})
```

### Step 2: Run test to verify it fails

Run: `docker compose exec nuxt npm run test -- UiAccordionClassCounters`
Expected: FAIL - Component does not exist

### Step 3: Write minimal implementation

**Create `app/components/ui/UiAccordionClassCounters.vue`:**

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
</script>

<template>
  <div
    v-if="counters && counters.length > 0"
    class="p-4"
  >
    <!-- Desktop Table -->
    <div class="hidden md:block overflow-x-auto">
      <table class="min-w-full divide-y divide-gray-200 dark:divide-gray-700">
        <thead class="bg-gray-50 dark:bg-gray-800">
          <tr>
            <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider">
              Level
            </th>
            <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider">
              Counter
            </th>
            <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider">
              Value
            </th>
            <th class="px-4 py-3 text-left text-xs font-medium text-gray-500 dark:text-gray-400 uppercase tracking-wider">
              Reset Timing
            </th>
          </tr>
        </thead>
        <tbody class="bg-white dark:bg-gray-900 divide-y divide-gray-200 dark:divide-gray-700">
          <tr
            v-for="counter in sortedCounters"
            :key="counter.id"
          >
            <td class="px-4 py-3 whitespace-nowrap text-sm text-gray-900 dark:text-gray-100">
              {{ counter.level }}
            </td>
            <td class="px-4 py-3 whitespace-nowrap text-sm font-medium text-gray-900 dark:text-gray-100">
              {{ counter.counter_name }}
            </td>
            <td class="px-4 py-3 whitespace-nowrap text-sm text-gray-900 dark:text-gray-100">
              {{ counter.counter_value }}
            </td>
            <td class="px-4 py-3 whitespace-nowrap text-sm">
              <UBadge
                :color="getResetTimingColor(counter.reset_timing)"
                variant="soft"
                size="sm"
              >
                {{ counter.reset_timing }}
              </UBadge>
            </td>
          </tr>
        </tbody>
      </table>
    </div>

    <!-- Mobile Stacked -->
    <div class="md:hidden space-y-4">
      <div
        v-for="counter in sortedCounters"
        :key="counter.id"
        class="p-4 rounded-lg bg-gray-50 dark:bg-gray-800 space-y-2"
      >
        <div class="flex justify-between items-center">
          <span class="text-sm font-medium text-gray-900 dark:text-gray-100">
            {{ counter.counter_name }}
          </span>
          <span class="text-sm text-gray-600 dark:text-gray-400">
            Level {{ counter.level }}
          </span>
        </div>
        <div class="flex justify-between items-center">
          <span class="text-sm text-gray-700 dark:text-gray-300">
            Value: {{ counter.counter_value }}
          </span>
          <UBadge
            :color="getResetTimingColor(counter.reset_timing)"
            variant="soft"
            size="sm"
          >
            {{ counter.reset_timing }}
          </UBadge>
        </div>
      </div>
    </div>
  </div>
</template>
```

### Step 4: Run test to verify it passes

Run: `docker compose exec nuxt npm run test -- UiAccordionClassCounters`
Expected: All tests PASS

### Step 5: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 6: Commit

```bash
git add app/components/ui/UiAccordionClassCounters.vue tests/components/ui/UiAccordionClassCounters.test.ts
git commit -m "feat: Add UiAccordionClassCounters component with tests

- Display class counters in responsive table format
- Sort by level ascending
- Color-coded reset timing badges
- Mobile-responsive stacked layout
- 5 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: UiAccordionItemDetail Component

**Files:**
- Create: `app/components/ui/UiAccordionItemDetail.vue`
- Create: `tests/components/ui/UiAccordionItemDetail.test.ts`

### Step 1: Write the failing test

**Create `tests/components/ui/UiAccordionItemDetail.test.ts`:**

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionItemDetail from '~/components/ui/UiAccordionItemDetail.vue'

describe('UiAccordionItemDetail', () => {
  it('renders detail text', async () => {
    const wrapper = await mountSuspended(UiAccordionItemDetail, {
      props: { detail: 'This is a magic sword forged in dragon fire.' }
    })

    expect(wrapper.text()).toContain('This is a magic sword forged in dragon fire.')
  })

  it('preserves whitespace for multi-line content', async () => {
    const multilineDetail = 'Line 1\nLine 2\nLine 3'
    const wrapper = await mountSuspended(UiAccordionItemDetail, {
      props: { detail: multilineDetail }
    })

    expect(wrapper.html()).toContain('whitespace-pre-line')
  })

  it('applies italic styling', async () => {
    const wrapper = await mountSuspended(UiAccordionItemDetail, {
      props: { detail: 'Flavor text' }
    })

    expect(wrapper.html()).toContain('italic')
  })

  it('does not render when detail is null', async () => {
    const wrapper = await mountSuspended(UiAccordionItemDetail, {
      props: { detail: null }
    })

    expect(wrapper.html()).toBe('<!--v-if-->')
  })

  it('does not render when detail is empty string', async () => {
    const wrapper = await mountSuspended(UiAccordionItemDetail, {
      props: { detail: '' }
    })

    expect(wrapper.html()).toBe('<!--v-if-->')
  })
})
```

### Step 2: Run test to verify it fails

Run: `docker compose exec nuxt npm run test -- UiAccordionItemDetail`
Expected: FAIL - Component does not exist

### Step 3: Write minimal implementation

**Create `app/components/ui/UiAccordionItemDetail.vue`:**

```vue
<script setup lang="ts">
interface Props {
  detail: string | null
}

const props = defineProps<Props>()
</script>

<template>
  <div
    v-if="detail"
    class="p-4"
  >
    <p class="text-sm text-gray-600 dark:text-gray-400 italic whitespace-pre-line leading-relaxed">
      {{ detail }}
    </p>
  </div>
</template>
```

### Step 4: Run test to verify it passes

Run: `docker compose exec nuxt npm run test -- UiAccordionItemDetail`
Expected: All tests PASS

### Step 5: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 6: Commit

```bash
git add app/components/ui/UiAccordionItemDetail.vue tests/components/ui/UiAccordionItemDetail.test.ts
git commit -m "feat: Add UiAccordionItemDetail component with tests

- Display item detail flavor text
- Preserve whitespace for multi-line content
- Italic styling to differentiate from main description
- Conditional rendering (null/empty check)
- 5 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: UiAccordionPrerequisites Component

**Files:**
- Create: `app/components/ui/UiAccordionPrerequisites.vue`
- Create: `tests/components/ui/UiAccordionPrerequisites.test.ts`

### Step 1: Write the failing test

**Create `tests/components/ui/UiAccordionPrerequisites.test.ts`:**

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionPrerequisites from '~/components/ui/UiAccordionPrerequisites.vue'
import type { EntityPrerequisiteResource } from '~/types/api/entities'

describe('UiAccordionPrerequisites', () => {
  it('renders ability score prerequisites', async () => {
    const prerequisites: EntityPrerequisiteResource[] = [
      {
        id: 1,
        prerequisite_type: 'ability_score',
        prerequisite_id: 1,
        minimum_value: 13,
        description: null,
        group_id: 1,
        ability_score: {
          id: 1,
          code: 'STR',
          name: 'Strength'
        }
      }
    ]

    const wrapper = await mountSuspended(UiAccordionPrerequisites, {
      props: { prerequisites }
    })

    expect(wrapper.text()).toContain('Strength 13 or higher')
  })

  it('renders race prerequisites', async () => {
    const prerequisites: EntityPrerequisiteResource[] = [
      {
        id: 2,
        prerequisite_type: 'race',
        prerequisite_id: 5,
        minimum_value: null,
        description: null,
        group_id: 1,
        race: {
          id: 5,
          slug: 'dwarf',
          name: 'Dwarf',
          speed: 25,
          size: { id: 1, code: 'M', name: 'Medium' }
        }
      }
    ]

    const wrapper = await mountSuspended(UiAccordionPrerequisites, {
      props: { prerequisites }
    })

    expect(wrapper.text()).toContain('Must be a Dwarf')
  })

  it('renders custom description prerequisites', async () => {
    const prerequisites: EntityPrerequisiteResource[] = [
      {
        id: 3,
        prerequisite_type: null,
        prerequisite_id: null,
        minimum_value: null,
        description: 'Attunement by a spellcaster',
        group_id: 1
      }
    ]

    const wrapper = await mountSuspended(UiAccordionPrerequisites, {
      props: { prerequisites }
    })

    expect(wrapper.text()).toContain('Attunement by a spellcaster')
  })

  it('handles empty prerequisites array', async () => {
    const wrapper = await mountSuspended(UiAccordionPrerequisites, {
      props: { prerequisites: [] }
    })

    expect(wrapper.text()).toBe('')
  })

  it('displays multiple prerequisites', async () => {
    const prerequisites: EntityPrerequisiteResource[] = [
      {
        id: 1,
        prerequisite_type: 'ability_score',
        prerequisite_id: 1,
        minimum_value: 13,
        description: null,
        group_id: 1,
        ability_score: { id: 1, code: 'STR', name: 'Strength' }
      },
      {
        id: 2,
        prerequisite_type: null,
        prerequisite_id: null,
        minimum_value: null,
        description: 'Proficient with martial weapons',
        group_id: 2
      }
    ]

    const wrapper = await mountSuspended(UiAccordionPrerequisites, {
      props: { prerequisites }
    })

    expect(wrapper.text()).toContain('Strength 13 or higher')
    expect(wrapper.text()).toContain('Proficient with martial weapons')
  })
})
```

### Step 2: Run test to verify it fails

Run: `docker compose exec nuxt npm run test -- UiAccordionPrerequisites`
Expected: FAIL - Component does not exist

### Step 3: Write minimal implementation

**Create `app/components/ui/UiAccordionPrerequisites.vue`:**

```vue
<script setup lang="ts">
import type { EntityPrerequisiteResource } from '~/types/api/entities'

interface Props {
  prerequisites: EntityPrerequisiteResource[]
}

const props = defineProps<Props>()

/**
 * Format prerequisite for display
 */
const formatPrerequisite = (prereq: EntityPrerequisiteResource): string => {
  // Custom description takes precedence
  if (prereq.description) {
    return prereq.description
  }

  // Ability score prerequisite
  if (prereq.ability_score) {
    return `${prereq.ability_score.name} ${prereq.minimum_value} or higher`
  }

  // Race prerequisite
  if (prereq.race) {
    return `Must be a ${prereq.race.name}`
  }

  // Skill prerequisite
  if (prereq.skill) {
    return `Proficient in ${prereq.skill.name}`
  }

  // Proficiency type prerequisite
  if (prereq.proficiency_type) {
    return `Proficient with ${prereq.proficiency_type.name}`
  }

  // Fallback
  return prereq.prerequisite_type || 'Unknown prerequisite'
}
</script>

<template>
  <div
    v-if="prerequisites && prerequisites.length > 0"
    class="p-4"
  >
    <ul class="space-y-2">
      <li
        v-for="prereq in prerequisites"
        :key="prereq.id"
        class="text-sm text-gray-700 dark:text-gray-300 flex items-start gap-2"
      >
        <span class="text-gray-400 dark:text-gray-600">•</span>
        <span>{{ formatPrerequisite(prereq) }}</span>
      </li>
    </ul>
  </div>
</template>
```

### Step 4: Run test to verify it passes

Run: `docker compose exec nuxt npm run test -- UiAccordionPrerequisites`
Expected: All tests PASS

### Step 5: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 6: Commit

```bash
git add app/components/ui/UiAccordionPrerequisites.vue tests/components/ui/UiAccordionPrerequisites.test.ts
git commit -m "feat: Add UiAccordionPrerequisites component with tests

- Display item prerequisites (ability scores, races, proficiencies)
- Smart formatting based on prerequisite type
- Bullet list layout
- 5 passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Update Classes Detail Page

**Files:**
- Modify: `app/pages/classes/[slug].vue`
- Modify: `tests/pages/classes/[slug].test.ts` (if exists)

### Step 1: Read current Classes page

Run: `cat app/pages/classes/[slug].vue`

### Step 2: Add counters accordion section

**In `app/pages/classes/[slug].vue`**, find the `UAccordion :items` array (around line 110) and add the counters section:

```vue
<!-- Before the existing accordion items, add this import at top of script -->
<script setup lang="ts">
// ... existing imports ...
</script>

<!-- In the template, update the accordion items array -->
<UAccordion
  :items="[
    ...(entity.counters && entity.counters.length > 0 ? [{
      label: 'Class Counters',
      slot: 'counters',
      defaultOpen: false
    }] : []),
    ...(remainingTraits.length > 0 ? [{
      label: `Additional Class Traits (${remainingTraits.length})`,
      slot: 'traits',
      defaultOpen: false
    }] : []),
    <!-- ... rest of existing items ... -->
  ]"
  type="multiple"
>
  <!-- Add counters slot BEFORE traits slot -->
  <template
    v-if="entity.counters && entity.counters.length > 0"
    #counters
  >
    <UiAccordionClassCounters :counters="entity.counters" />
  </template>

  <!-- ... rest of existing slots ... -->
</UAccordion>
```

### Step 3: Verify page renders

Run: `docker compose exec nuxt npm run dev`
Visit: `http://localhost:3000/classes/barbarian` (or any class with counters)
Expected: Counters accordion appears (if data exists)

### Step 4: Run tests

Run: `docker compose exec nuxt npm run test`
Expected: All existing tests still pass

### Step 5: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 6: Commit

```bash
git add app/pages/classes/[slug].vue
git commit -m "feat: Add class counters display to Classes detail page

- Add UiAccordionClassCounters to accordion sections
- Conditional rendering based on data presence
- Positioned before traits section

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Update Feats Detail Page

**Files:**
- Modify: `app/pages/feats/[slug].vue`

### Step 1: Read current Feats page

Run: `cat app/pages/feats/[slug].vue`

### Step 2: Add proficiencies and conditions accordion sections

**In `app/pages/feats/[slug].vue`**, update the accordion items array (around line 120):

```vue
<UAccordion
  :items="[
    ...(entity.proficiencies && entity.proficiencies.length > 0 ? [{
      label: 'Proficiencies',
      slot: 'proficiencies',
      defaultOpen: false
    }] : []),
    ...(entity.conditions && entity.conditions.length > 0 ? [{
      label: 'Conditions',
      slot: 'conditions',
      defaultOpen: false
    }] : []),
    ...(entity.modifiers && entity.modifiers.length > 0 ? [{
      label: 'Modifiers',
      slot: 'modifiers',
      defaultOpen: false
    }] : []),
    <!-- ... rest of existing items ... -->
  ]"
  type="multiple"
>
  <!-- Add proficiencies slot -->
  <template
    v-if="entity.proficiencies && entity.proficiencies.length > 0"
    #proficiencies
  >
    <UiAccordionBulletList :items="entity.proficiencies" />
  </template>

  <!-- Add conditions slot -->
  <template
    v-if="entity.conditions && entity.conditions.length > 0"
    #conditions
  >
    <UiAccordionConditions
      :conditions="entity.conditions"
      entity-type="feat"
    />
  </template>

  <!-- ... rest of existing slots ... -->
</UAccordion>
```

### Step 3: Verify page renders

Run: `docker compose exec nuxt npm run dev`
Visit: `http://localhost:3000/feats/skilled` (or any feat)
Expected: Proficiencies and conditions accordions appear (if data exists)

### Step 4: Run tests

Run: `docker compose exec nuxt npm run test`
Expected: All existing tests still pass

### Step 5: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 6: Commit

```bash
git add app/pages/feats/[slug].vue
git commit -m "feat: Add proficiencies and conditions to Feats detail page

- Add proficiencies accordion using UiAccordionBulletList
- Add conditions accordion using UiAccordionConditions
- Conditional rendering based on data presence

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 8: Update Monsters Detail Page

**Files:**
- Modify: `app/pages/monsters/[slug].vue`

### Step 1: Read current Monsters page

Run: `cat app/pages/monsters/[slug].vue`

### Step 2: Add conditions accordion after Modifiers

**In `app/pages/monsters/[slug].vue`**, find the Modifiers section (around line 230) and add conditions after it:

```vue
<!-- After Modifiers section -->
<UiModifiersDisplay
  v-if="monster.modifiers && monster.modifiers.length > 0"
  :modifiers="monster.modifiers"
/>

<!-- Add Conditions section -->
<UCard v-if="monster.conditions && monster.conditions.length > 0">
  <template #header>
    <h2 class="text-lg font-semibold text-gray-900 dark:text-gray-100">
      Conditions
    </h2>
  </template>
  <UiAccordionConditions
    :conditions="monster.conditions"
    entity-type="monster"
  />
</UCard>

<!-- Existing Sources section below -->
<UiSourceDisplay
  v-if="monster.sources && monster.sources.length > 0"
  :sources="monster.sources"
/>
```

### Step 3: Verify page renders

Run: `docker compose exec nuxt npm run dev`
Visit: `http://localhost:3000/monsters/zombie` (or any monster)
Expected: Conditions section appears (if data exists)

### Step 4: Run tests

Run: `docker compose exec nuxt npm run test`
Expected: All existing tests still pass

### Step 5: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 6: Commit

```bash
git add app/pages/monsters/[slug].vue
git commit -m "feat: Add conditions display to Monsters detail page

- Add conditions section using UiAccordionConditions
- Positioned between modifiers and sources
- Conditional rendering based on data presence

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 9: Update Items Detail Page

**Files:**
- Modify: `app/pages/items/[slug].vue`

### Step 1: Read current Items page

Run: `cat app/pages/items/[slug].vue`

### Step 2: Add detail and prerequisites accordion sections

**In `app/pages/items/[slug].vue`**, update the accordion items array (around line 138):

```vue
<UAccordion
  :items="[
    ...(item.detail ? [{
      label: 'Additional Details',
      slot: 'detail',
      defaultOpen: false
    }] : []),
    ...(item.prerequisites && item.prerequisites.length > 0 ? [{
      label: 'Prerequisites',
      slot: 'prerequisites',
      defaultOpen: false
    }] : []),
    ...(item.properties && item.properties.length > 0 ? [{
      label: 'Properties',
      slot: 'properties',
      defaultOpen: false
    }] : []),
    <!-- ... rest of existing items ... -->
  ]"
  type="multiple"
>
  <!-- Add detail slot -->
  <template
    v-if="item.detail"
    #detail
  >
    <UiAccordionItemDetail :detail="item.detail" />
  </template>

  <!-- Add prerequisites slot -->
  <template
    v-if="item.prerequisites && item.prerequisites.length > 0"
    #prerequisites
  >
    <UiAccordionPrerequisites :prerequisites="item.prerequisites" />
  </template>

  <!-- ... rest of existing slots ... -->
</UAccordion>
```

### Step 3: Verify page renders

Run: `docker compose exec nuxt npm run dev`
Visit: `http://localhost:3000/items/staff-of-the-magi` (or any magic item)
Expected: Detail and prerequisites accordions appear (if data exists)

### Step 4: Run tests

Run: `docker compose exec nuxt npm run test`
Expected: All existing tests still pass

### Step 5: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 6: Commit

```bash
git add app/pages/items/[slug].vue
git commit -m "feat: Add detail and prerequisites to Items detail page

- Add detail accordion using UiAccordionItemDetail
- Add prerequisites accordion using UiAccordionPrerequisites
- Conditional rendering based on data presence

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 10: Standardize Backgrounds Image Component

**Files:**
- Modify: `app/pages/backgrounds/[slug].vue`

### Step 1: Read current Backgrounds page

Run: `cat app/pages/backgrounds/[slug].vue`

### Step 2: Replace raw NuxtImg with UiDetailEntityImage

**In `app/pages/backgrounds/[slug].vue`**, find the image section (around lines 80-93) and replace:

**OLD:**
```vue
<div
  v-if="imagePath"
  class="lg:w-1/3 flex-shrink-0"
>
  <NuxtImg
    :src="imagePath"
    :alt="`${entity.name} background illustration`"
    class="w-full h-auto rounded-lg shadow-lg object-cover"
    loading="lazy"
    width="512"
    height="512"
  />
</div>
```

**NEW:**
```vue
<div
  v-if="imagePath"
  class="lg:w-1/3 flex-shrink-0"
>
  <UiDetailEntityImage
    :image-path="imagePath"
    :image-alt="`${entity.name} background illustration`"
  />
</div>
```

### Step 3: Verify page renders

Run: `docker compose exec nuxt npm run dev`
Visit: `http://localhost:3000/backgrounds/acolyte`
Expected: Image displays correctly with standardized component

### Step 4: Run tests

Run: `docker compose exec nuxt npm run test`
Expected: All existing tests still pass

### Step 5: Verify TypeScript compiles

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 6: Commit

```bash
git add app/pages/backgrounds/[slug].vue
git commit -m "refactor: Standardize image component in Backgrounds page

- Replace raw NuxtImg with UiDetailEntityImage component
- Maintains same layout and styling
- Consistent with other detail pages

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 11: Final Verification & CHANGELOG

**Files:**
- Modify: `CHANGELOG.md`

### Step 1: Run full test suite

Run: `docker compose exec nuxt npm run test`
Expected: All tests PASS

### Step 2: Verify TypeScript compilation

Run: `docker compose exec nuxt npm run typecheck`
Expected: No errors

### Step 3: Run linter

Run: `docker compose exec nuxt npm run lint`
Expected: No errors

### Step 4: Browser testing checklist

Test in browser:
- [ ] Classes page shows counters (visit `/classes/barbarian`)
- [ ] Feats page shows proficiencies and conditions (visit `/feats/skilled`)
- [ ] Monsters page shows conditions (visit `/monsters/zombie`)
- [ ] Items page shows detail and prerequisites (visit `/items/staff-of-the-magi`)
- [ ] Backgrounds page image renders correctly (visit `/backgrounds/acolyte`)
- [ ] Dark mode works for all new components
- [ ] Mobile responsive (test at 375px width)
- [ ] All accordions expand/collapse correctly

### Step 5: Update CHANGELOG.md

**Add to the top of `CHANGELOG.md`:**

```markdown
### Added
- Class counters display (resource tracking like Rage, Ki Points) with level progression table (2025-11-23)
- Feat proficiencies display for proficiency-granting feats (2025-11-23)
- Conditions display for Feats and Monsters (immunities, resistances, vulnerabilities) (2025-11-23)
- Item detail field display for flavor text and usage notes (2025-11-23)
- Item prerequisites display for usage requirements (2025-11-23)
- UiAccordionConditions reusable component (2025-11-23)
- UiAccordionClassCounters reusable component (2025-11-23)
- UiAccordionItemDetail reusable component (2025-11-23)
- UiAccordionPrerequisites reusable component (2025-11-23)
- Color utilities for reset timing and condition effect badges (2025-11-23)

### Changed
- Standardized Backgrounds page to use UiDetailEntityImage component (2025-11-23)
- Improved consistency of conditions display across Races, Feats, and Monsters (2025-11-23)

### Fixed
- Missing API fields now displayed across all 7 detail pages (2025-11-23)
```

### Step 6: Final commit

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG for detail pages missing fields

- 4 new reusable components
- 10 new features across 5 detail pages
- 1 standardization improvement
- 21 new passing tests

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Success Criteria

All tasks complete when:
- ✅ 4 new components created with comprehensive tests
- ✅ 5 detail pages updated
- ✅ All tests passing (21+ new tests)
- ✅ TypeScript compiles without errors
- ✅ ESLint passes
- ✅ Browser verification complete (light/dark mode, responsive)
- ✅ CHANGELOG.md updated
- ✅ All work committed

---

## Execution Notes

**Estimated Time:** 90-120 minutes (following TDD)

**Key Dependencies:**
- Docker must be running (`docker compose up -d`)
- Backend API must be running at `localhost:8080`
- Node modules installed (`docker compose exec nuxt npm install`)

**Testing Strategy:**
- Write test first (RED)
- Implement minimal code (GREEN)
- Refactor if needed
- Commit immediately

**Rollback Plan:**
If any task fails critically:
```bash
git log --oneline -n 10  # Find last good commit
git reset --hard <commit-hash>
```
