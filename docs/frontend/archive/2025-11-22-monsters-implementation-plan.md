# Monsters Feature Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Monsters as the 7th entity type with list/detail pages, CR/Type filtering, and reusable accordion components.

**Architecture:** Follow existing entity patterns (spells/items), create 3 new components (MonsterCard, UiAccordionActions, UiAccordionTraits), reuse existing composables and test helpers.

**Tech Stack:** Nuxt 4.x, NuxtUI 4.x, TypeScript, Vitest, OpenAPI types

---

## Task 1: Add Monster Type Export

**Files:**
- Modify: `app/types/api/entities.ts`

**Step 1: Add Monster type export**

```typescript
// Add after existing type exports
export type Monster = components['schemas']['MonsterResource']
```

**Step 2: Verify type exists in generated types**

Run: `grep -n "MonsterResource" app/types/api/generated.ts`
Expected: Should find MonsterResource schema definition

**Step 3: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck 2>&1 | grep -c "error TS"`
Expected: 13 errors (no regression)

**Step 4: Commit**

```bash
git add app/types/api/entities.ts
git commit -m "feat: Add Monster type export from OpenAPI schema

- Export Monster type from generated MonsterResource
- Enables type-safe monster components

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Add Challenge Rating Badge Color Utility

**Files:**
- Modify: `app/utils/badgeColors.ts`
- Modify: `tests/utils/badgeColors.test.ts`

**Step 1: Write failing tests for getChallengeRatingColor**

Add to `tests/utils/badgeColors.test.ts`:

```typescript
describe('getChallengeRatingColor', () => {
  it('returns success for CR 0 (easy tier)', () => {
    expect(getChallengeRatingColor('0')).toBe('success')
  })

  it('returns success for CR 1/8 (fractional easy)', () => {
    expect(getChallengeRatingColor('1/8')).toBe('success')
  })

  it('returns success for CR 1/4', () => {
    expect(getChallengeRatingColor('1/4')).toBe('success')
  })

  it('returns success for CR 4 (easy tier max)', () => {
    expect(getChallengeRatingColor('4')).toBe('success')
  })

  it('returns info for CR 5 (medium tier)', () => {
    expect(getChallengeRatingColor('5')).toBe('info')
  })

  it('returns info for CR 10 (medium tier max)', () => {
    expect(getChallengeRatingColor('10')).toBe('info')
  })

  it('returns warning for CR 11 (hard tier)', () => {
    expect(getChallengeRatingColor('11')).toBe('warning')
  })

  it('returns warning for CR 16 (hard tier max)', () => {
    expect(getChallenageRatingColor('16')).toBe('warning')
  })

  it('returns error for CR 17 (deadly tier)', () => {
    expect(getChallengeRatingColor('17')).toBe('error')
  })

  it('returns error for CR 30 (max CR)', () => {
    expect(getChallengeRatingColor('30')).toBe('error')
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm test -- badgeColors.test.ts`
Expected: 10 new failing tests (getChallengeRatingColor not defined)

**Step 3: Implement getChallengeRatingColor function**

Add to `app/utils/badgeColors.ts`:

```typescript
/**
 * Get badge color for Challenge Rating
 * Maps CR to D&D difficulty tiers:
 * - CR 0-4: Easy (success/green)
 * - CR 5-10: Medium (info/blue)
 * - CR 11-16: Hard (warning/yellow)
 * - CR 17+: Deadly (error/red)
 */
export function getChallengeRatingColor(cr: string): BadgeColor {
  // CR can be fractional like "1/8", "1/4", "1/2" or whole numbers "1" through "30"
  const numericCR = cr.includes('/')
    ? eval(cr) as number // "1/4" evaluates to 0.25
    : parseFloat(cr)

  if (numericCR <= 4) return 'success'  // Easy
  if (numericCR <= 10) return 'info'    // Medium
  if (numericCR <= 16) return 'warning' // Hard
  return 'error'                         // Deadly
}
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm test -- badgeColors.test.ts`
Expected: All badge color tests pass (18 existing + 10 new = 28 total)

**Step 5: Commit**

```bash
git add app/utils/badgeColors.ts tests/utils/badgeColors.test.ts
git commit -m "feat: Add getChallengeRatingColor utility function

- Maps CR to D&D difficulty tiers (easy/medium/hard/deadly)
- Handles fractional CRs (1/8, 1/4, 1/2)
- 10 new tests added (all passing)
- Tests: 574/574 passing ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Create UiAccordionTraits Component

**Files:**
- Create: `app/components/ui/accordion/UiAccordionTraits.vue`
- Create: `tests/components/ui/accordion/UiAccordionTraits.test.ts`

**Step 1: Write failing tests for UiAccordionTraits**

Create `tests/components/ui/accordion/UiAccordionTraits.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionTraits from '~/components/ui/accordion/UiAccordionTraits.vue'

describe('UiAccordionTraits', () => {
  const mockTraits = [
    {
      id: 1,
      name: 'Legendary Resistance (3/Day)',
      description: 'If the dragon fails a saving throw, it can choose to succeed instead.'
    },
    {
      id: 2,
      name: 'Amphibious',
      description: 'The creature can breathe air and water.'
    }
  ]

  it('renders accordion header with trait count', async () => {
    const wrapper = await mountSuspended(UiAccordionTraits, {
      props: { traits: mockTraits }
    })

    expect(wrapper.text()).toContain('Traits (2)')
  })

  it('uses default title "Traits"', async () => {
    const wrapper = await mountSuspended(UiAccordionTraits, {
      props: { traits: mockTraits }
    })

    expect(wrapper.text()).toContain('Traits')
  })

  it('uses custom title when provided', async () => {
    const wrapper = await mountSuspended(UiAccordionTraits, {
      props: {
        traits: mockTraits,
        title: 'Special Abilities'
      }
    })

    expect(wrapper.text()).toContain('Special Abilities (2)')
  })

  it('displays trait names', async () => {
    const wrapper = await mountSuspended(UiAccordionTraits, {
      props: { traits: mockTraits }
    })

    expect(wrapper.text()).toContain('Legendary Resistance (3/Day)')
    expect(wrapper.text()).toContain('Amphibious')
  })

  it('displays trait descriptions', async () => {
    const wrapper = await mountSuspended(UiAccordionTraits, {
      props: { traits: mockTraits }
    })

    expect(wrapper.text()).toContain('If the dragon fails a saving throw')
    expect(wrapper.text()).toContain('The creature can breathe air and water')
  })

  it('handles empty traits array', async () => {
    const wrapper = await mountSuspended(UiAccordionTraits, {
      props: { traits: [] }
    })

    expect(wrapper.text()).toContain('Traits (0)')
  })

  it('does not render when no traits provided', async () => {
    const wrapper = await mountSuspended(UiAccordionTraits, {
      props: { traits: [] }
    })

    // Component renders but shows empty state
    const accordion = wrapper.find('[data-testid="accordion"]')
    expect(accordion.exists()).toBe(false)
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm test -- UiAccordionTraits.test.ts`
Expected: All 7 tests fail (component doesn't exist)

**Step 3: Create UiAccordionTraits component**

Create `app/components/ui/accordion/UiAccordionTraits.vue`:

```vue
<script setup lang="ts">
interface Trait {
  id?: number
  name: string
  description: string
}

interface Props {
  traits: Trait[]
  title?: string
}

const props = withDefaults(defineProps<Props>(), {
  title: 'Traits'
})

const items = computed(() => {
  if (!props.traits || props.traits.length === 0) return []

  return [{
    label: `${props.title} (${props.traits.length})`,
    defaultOpen: true,
    slot: 'traits'
  }]
})
</script>

<template>
  <UAccordion
    v-if="traits.length > 0"
    :items="items"
    :ui="{
      item: {
        base: 'border border-gray-200 dark:border-gray-700 rounded-lg overflow-hidden mb-4'
      }
    }"
  >
    <template #traits>
      <div class="space-y-4 p-4">
        <div
          v-for="trait in traits"
          :key="trait.id || trait.name"
          class="space-y-1"
        >
          <h4 class="font-semibold text-primary-600 dark:text-primary-400">
            {{ trait.name }}
          </h4>
          <p class="text-sm text-gray-700 dark:text-gray-300">
            {{ trait.description }}
          </p>
        </div>
      </div>
    </template>
  </UAccordion>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm test -- UiAccordionTraits.test.ts`
Expected: All 7 tests pass

**Step 5: Commit**

```bash
git add app/components/ui/accordion/UiAccordionTraits.vue tests/components/ui/accordion/UiAccordionTraits.test.ts
git commit -m "feat: Add UiAccordionTraits component (TDD)

- Reusable accordion for displaying passive traits
- Supports custom title prop
- 7 tests added (all passing)
- Tests: 581/581 passing ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Create UiAccordionActions Component

**Files:**
- Create: `app/components/ui/accordion/UiAccordionActions.vue`
- Create: `tests/components/ui/accordion/UiAccordionActions.test.ts`

**Step 1: Write failing tests for UiAccordionActions**

Create `tests/components/ui/accordion/UiAccordionActions.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UiAccordionActions from '~/components/ui/accordion/UiAccordionActions.vue'

describe('UiAccordionActions', () => {
  const mockActions = [
    {
      id: 1,
      name: 'Multiattack',
      description: 'The dragon can use its Frightful Presence. It then makes three attacks.',
      attack_data: null,
      recharge: null
    },
    {
      id: 2,
      name: 'Fire Breath',
      description: 'The dragon exhales fire in a 90-foot cone.',
      attack_data: '["Fire Damage||26d6"]',
      recharge: '5-6'
    }
  ]

  const mockLegendaryActions = [
    {
      id: 1,
      name: 'Detect',
      description: 'The dragon makes a Wisdom (Perception) check.',
      action_cost: 1,
      attack_data: null,
      recharge: null
    },
    {
      id: 2,
      name: 'Wing Attack',
      description: 'The dragon beats its wings.',
      action_cost: 2,
      attack_data: '["Bludgeoning Damage||2d6+10"]',
      recharge: null
    }
  ]

  it('renders accordion header with action count', async () => {
    const wrapper = await mountSuspended(UiAccordionActions, {
      props: {
        actions: mockActions,
        title: 'Actions'
      }
    })

    expect(wrapper.text()).toContain('Actions (2)')
  })

  it('displays action names', async () => {
    const wrapper = await mountSuspended(UiAccordionActions, {
      props: {
        actions: mockActions,
        title: 'Actions'
      }
    })

    expect(wrapper.text()).toContain('Multiattack')
    expect(wrapper.text()).toContain('Fire Breath')
  })

  it('displays action descriptions', async () => {
    const wrapper = await mountSuspended(UiAccordionActions, {
      props: {
        actions: mockActions,
        title: 'Actions'
      }
    })

    expect(wrapper.text()).toContain('The dragon can use its Frightful Presence')
    expect(wrapper.text()).toContain('The dragon exhales fire')
  })

  it('shows recharge badge when present', async () => {
    const wrapper = await mountSuspended(UiAccordionActions, {
      props: {
        actions: mockActions,
        title: 'Actions'
      }
    })

    expect(wrapper.text()).toContain('Recharge 5-6')
  })

  it('does not show recharge badge when null', async () => {
    const wrapper = await mountSuspended(UiAccordionActions, {
      props: {
        actions: [mockActions[0]], // Only Multiattack (no recharge)
        title: 'Actions'
      }
    })

    expect(wrapper.text()).not.toContain('Recharge')
  })

  it('shows action cost badge when showCost is true', async () => {
    const wrapper = await mountSuspended(UiAccordionActions, {
      props: {
        actions: mockLegendaryActions,
        title: 'Legendary Actions',
        showCost: true
      }
    })

    expect(wrapper.text()).toContain('Costs 1 Action')
    expect(wrapper.text()).toContain('Costs 2 Actions')
  })

  it('does not show action cost when showCost is false', async () => {
    const wrapper = await mountSuspended(UiAccordionActions, {
      props: {
        actions: mockLegendaryActions,
        title: 'Legendary Actions',
        showCost: false
      }
    })

    expect(wrapper.text()).not.toContain('Costs')
  })

  it('handles empty actions array', async () => {
    const wrapper = await mountSuspended(UiAccordionActions, {
      props: {
        actions: [],
        title: 'Actions'
      }
    })

    expect(wrapper.text()).toContain('Actions (0)')
  })

  it('does not render when no actions provided', async () => {
    const wrapper = await mountSuspended(UiAccordionActions, {
      props: {
        actions: [],
        title: 'Actions'
      }
    })

    const accordion = wrapper.find('[data-testid="accordion"]')
    expect(accordion.exists()).toBe(false)
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm test -- UiAccordionActions.test.ts`
Expected: All 9 tests fail (component doesn't exist)

**Step 3: Create UiAccordionActions component**

Create `app/components/ui/accordion/UiAccordionActions.vue`:

```vue
<script setup lang="ts">
interface Action {
  id?: number
  name: string
  description: string
  attack_data?: string | null
  recharge?: string | null
  action_cost?: number
}

interface Props {
  actions: Action[]
  title: string
  showCost?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  showCost: false
})

const items = computed(() => {
  if (!props.actions || props.actions.length === 0) return []

  return [{
    label: `${props.title} (${props.actions.length})`,
    defaultOpen: true,
    slot: 'actions'
  }]
})

/**
 * Get action cost text
 */
function getActionCostText(cost: number): string {
  return cost === 1 ? 'Costs 1 Action' : `Costs ${cost} Actions`
}
</script>

<template>
  <UAccordion
    v-if="actions.length > 0"
    :items="items"
    :ui="{
      item: {
        base: 'border border-gray-200 dark:border-gray-700 rounded-lg overflow-hidden mb-4'
      }
    }"
  >
    <template #actions>
      <div class="space-y-4 p-4">
        <div
          v-for="action in actions"
          :key="action.id || action.name"
          class="space-y-2"
        >
          <!-- Action name with badges -->
          <div class="flex items-center gap-2 flex-wrap">
            <h4 class="font-semibold text-primary-600 dark:text-primary-400">
              {{ action.name }}
            </h4>
            <UBadge
              v-if="action.recharge"
              color="warning"
              variant="soft"
              size="sm"
            >
              Recharge {{ action.recharge }}
            </UBadge>
            <UBadge
              v-if="showCost && action.action_cost"
              color="info"
              variant="soft"
              size="sm"
            >
              {{ getActionCostText(action.action_cost) }}
            </UBadge>
          </div>

          <!-- Action description -->
          <p class="text-sm text-gray-700 dark:text-gray-300">
            {{ action.description }}
          </p>
        </div>
      </div>
    </template>
  </UAccordion>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm test -- UiAccordionActions.test.ts`
Expected: All 9 tests pass

**Step 5: Commit**

```bash
git add app/components/ui/accordion/UiAccordionActions.vue tests/components/ui/accordion/UiAccordionActions.test.ts
git commit -m "feat: Add UiAccordionActions component (TDD)

- Reusable accordion for displaying actions/legendary actions
- Shows recharge badges and action costs
- 9 tests added (all passing)
- Tests: 590/590 passing ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Create MonsterCard Component

**Files:**
- Create: `app/components/monster/MonsterCard.vue`
- Create: `tests/components/monster/MonsterCard.test.ts`

**Step 1: Write failing tests for MonsterCard**

Create `tests/components/monster/MonsterCard.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import type { Monster } from '~/types'
import MonsterCard from '~/components/monster/MonsterCard.vue'
import { testCardLinkBehavior, testCardHoverEffects, testCardBorderStyling } from '../../helpers/cardBehavior'
import { testSourceFooter } from '../../helpers/sourceBehavior'

describe('MonsterCard', () => {
  const mockMonster: Monster = {
    id: 1,
    slug: 'ancient-red-dragon',
    name: 'Ancient Red Dragon',
    size: { id: 6, code: 'G', name: 'Gargantuan' },
    type: 'dragon',
    alignment: 'Chaotic Evil',
    armor_class: 22,
    armor_type: 'natural armor',
    hit_points_average: 546,
    hit_dice: '28d20+252',
    speed_walk: 40,
    speed_fly: 80,
    speed_swim: null,
    speed_burrow: null,
    speed_climb: 40,
    can_hover: false,
    strength: 30,
    dexterity: 10,
    constitution: 29,
    intelligence: 18,
    wisdom: 15,
    charisma: 23,
    challenge_rating: '24',
    experience_points: 62000,
    description: 'The most covetous of the true dragons, red dragons tirelessly seek to increase their treasure hoards.',
    traits: [],
    actions: [],
    legendary_actions: [
      { id: 1, name: 'Detect', description: 'The dragon makes a check.' }
    ],
    modifiers: [],
    conditions: [],
    sources: [
      { id: 1, code: 'MM', name: 'Monster Manual', pages: 'p. 97' }
    ]
  }

  // Shared behavior tests
  const mountCard = () => mountSuspended(MonsterCard, { props: { monster: mockMonster } })

  testCardLinkBehavior(mountCard, '/monsters/ancient-red-dragon')
  testCardHoverEffects(mountCard)
  testCardBorderStyling(mountCard)
  testSourceFooter(mountCard)

  it('renders monster name', async () => {
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster: mockMonster }
    })

    expect(wrapper.text()).toContain('Ancient Red Dragon')
  })

  it('renders CR badge with correct color', async () => {
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster: mockMonster }
    })

    expect(wrapper.text()).toContain('CR 24')
    // CR 24 should use error color (deadly tier)
    const badge = wrapper.find('[class*="bg-error"]')
    expect(badge.exists()).toBe(true)
  })

  it('renders type badge', async () => {
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster: mockMonster }
    })

    expect(wrapper.text()).toContain('dragon')
  })

  it('displays size', async () => {
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster: mockMonster }
    })

    expect(wrapper.text()).toContain('Gargantuan')
  })

  it('displays alignment', async () => {
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster: mockMonster }
    })

    expect(wrapper.text()).toContain('Chaotic Evil')
  })

  it('displays armor class', async () => {
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster: mockMonster }
    })

    expect(wrapper.text()).toContain('AC 22')
  })

  it('displays hit points', async () => {
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster: mockMonster }
    })

    expect(wrapper.text()).toContain('546 HP')
  })

  it('shows legendary indicator when has legendary actions', async () => {
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster: mockMonster }
    })

    expect(wrapper.text()).toContain('Legendary')
  })

  it('hides legendary indicator when no legendary actions', async () => {
    const monster = { ...mockMonster, legendary_actions: [] }
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster }
    })

    expect(wrapper.text()).not.toContain('Legendary')
  })

  it('truncates long descriptions', async () => {
    const longDesc = 'A'.repeat(200)
    const monster = { ...mockMonster, description: longDesc }
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster }
    })

    const text = wrapper.text()
    expect(text.length).toBeLessThan(200)
    expect(text).toContain('...')
  })

  it('does not truncate short descriptions', async () => {
    const shortDesc = 'A small creature.'
    const monster = { ...mockMonster, description: shortDesc }
    const wrapper = await mountSuspended(MonsterCard, {
      props: { monster }
    })

    expect(wrapper.text()).toContain(shortDesc)
    expect(wrapper.text()).not.toContain('...')
  })

  it('handles different CR values correctly', async () => {
    const crTests = [
      { cr: '0', expected: 'success' },
      { cr: '1/4', expected: 'success' },
      { cr: '5', expected: 'info' },
      { cr: '11', expected: 'warning' },
      { cr: '24', expected: 'error' }
    ]

    for (const test of crTests) {
      const monster = { ...mockMonster, challenge_rating: test.cr }
      const wrapper = await mountSuspended(MonsterCard, {
        props: { monster }
      })

      expect(wrapper.text()).toContain(`CR ${test.cr}`)
    }
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm test -- MonsterCard.test.ts`
Expected: All tests fail (component doesn't exist)

**Step 3: Create MonsterCard component**

Create `app/components/monster/MonsterCard.vue`:

```vue
<script setup lang="ts">
import type { Monster } from '~/types'
import { getChallengeRatingColor } from '~/utils/badgeColors'

interface Props {
  monster: Monster
}

const props = defineProps<Props>()

/**
 * Truncate description to specified length
 */
const truncatedDescription = computed(() => {
  const maxLength = 150
  if (!props.monster.description) return 'A creature from the D&D universe.'
  if (props.monster.description.length <= maxLength) return props.monster.description
  return props.monster.description.substring(0, maxLength).trim() + '...'
})

/**
 * Check if monster has legendary actions
 */
const isLegendary = computed(() => {
  return props.monster.legendary_actions && props.monster.legendary_actions.length > 0
})

/**
 * Get CR badge color
 */
const crBadgeColor = computed(() => {
  return getChallengeRatingColor(props.monster.challenge_rating)
})
</script>

<template>
  <NuxtLink
    :to="`/monsters/${monster.slug}`"
    class="block h-full"
  >
    <UCard class="hover:shadow-lg transition-shadow h-full border border-gray-200 dark:border-gray-700">
      <div class="flex flex-col h-full">
        <!-- Top content -->
        <div class="space-y-3 flex-1">
          <!-- CR and Type Badges -->
          <div class="flex items-center gap-2 flex-wrap justify-between">
            <UBadge
              :color="crBadgeColor"
              variant="subtle"
              size="md"
            >
              CR {{ monster.challenge_rating }}
            </UBadge>
            <UBadge
              color="neutral"
              variant="subtle"
              size="md"
            >
              {{ monster.type }}
            </UBadge>
          </div>

          <!-- Monster Name -->
          <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100 line-clamp-2">
            {{ monster.name }}
          </h3>

          <!-- Quick Stats -->
          <div class="space-y-1 text-sm text-gray-600 dark:text-gray-400">
            <div class="flex items-center gap-2">
              <span>🔷 {{ monster.size.name }}</span>
              <span>•</span>
              <span>{{ monster.alignment }}</span>
            </div>
            <div class="flex items-center gap-3">
              <span>⚔️ AC {{ monster.armor_class }}</span>
              <span>❤️ {{ monster.hit_points_average }} HP</span>
            </div>
            <div v-if="isLegendary">
              <UBadge
                color="warning"
                variant="soft"
                size="sm"
              >
                ⭐ Legendary
              </UBadge>
            </div>
          </div>

          <!-- Description Preview -->
          <p class="text-sm text-gray-700 dark:text-gray-300 line-clamp-3">
            {{ truncatedDescription }}
          </p>
        </div>

        <UiCardSourceFooter :sources="monster.sources" />
      </div>
    </UCard>
  </NuxtLink>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm test -- MonsterCard.test.ts`
Expected: All MonsterCard tests pass

**Step 5: Run full test suite**

Run: `docker compose exec nuxt npm test 2>&1 | tail -n 10`
Expected: All tests pass (590 + ~15 MonsterCard = ~605 tests)

**Step 6: Commit**

```bash
git add app/components/monster/MonsterCard.vue tests/components/monster/MonsterCard.test.ts
git commit -m "feat: Add MonsterCard component (TDD)

- Card component for monster list page
- Shows CR badge (color-coded), type, stats, legendary indicator
- Truncates long descriptions (150 chars max)
- Reuses test helpers for hover/border/source/link behavior
- 15+ tests added (all passing)
- Tests: ~605/605 passing ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 6: Create Monsters List Page

**Files:**
- Create: `app/pages/monsters/index.vue`

**Step 1: Create monsters list page**

Create `app/pages/monsters/index.vue`:

```vue
<script setup lang="ts">
import { ref, computed } from 'vue'
import type { Monster } from '~/types'

const route = useRoute()
const { apiFetch } = useApi()

// Custom filter state
const selectedCR = ref(route.query.cr ? String(route.query.cr) : null)
const selectedType = ref(route.query.type ? String(route.query.type) : null)

// CR range options
const crOptions = [
  { label: 'All CRs', value: null },
  { label: 'CR 0-4 (Easy)', value: '0-4' },
  { label: 'CR 5-10 (Medium)', value: '5-10' },
  { label: 'CR 11-16 (Hard)', value: '11-16' },
  { label: 'CR 17+ (Deadly)', value: '17+' }
]

// Type options (common D&D monster types)
const typeOptions = [
  { label: 'All Types', value: null },
  { label: 'Aberration', value: 'aberration' },
  { label: 'Beast', value: 'beast' },
  { label: 'Celestial', value: 'celestial' },
  { label: 'Construct', value: 'construct' },
  { label: 'Dragon', value: 'dragon' },
  { label: 'Elemental', value: 'elemental' },
  { label: 'Fey', value: 'fey' },
  { label: 'Fiend', value: 'fiend' },
  { label: 'Giant', value: 'giant' },
  { label: 'Humanoid', value: 'humanoid' },
  { label: 'Monstrosity', value: 'monstrosity' },
  { label: 'Ooze', value: 'ooze' },
  { label: 'Plant', value: 'plant' },
  { label: 'Undead', value: 'undead' }
]

// Query builder for custom filters
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  if (selectedCR.value) params.cr = selectedCR.value
  if (selectedType.value) params.type = selectedType.value
  return params
})

// Use entity list composable
const {
  data: monsters,
  pagination,
  loading,
  error,
  currentPage,
  search
} = useEntityList<Monster>({
  endpoint: '/monsters',
  cacheKey: 'monsters',
  defaultPerPage: 15,
  queryBuilder
})

// SEO
useSeoMeta({
  title: 'Monsters - D&D 5e Compendium',
  description: 'Browse D&D 5e monsters with stats, abilities, and lore. Filter by Challenge Rating and creature type.'
})

useHead({
  title: 'Monsters - D&D 5e Compendium'
})
</script>

<template>
  <div class="container mx-auto px-4 py-8">
    <!-- Page Header -->
    <UiListPageHeader
      title="Monsters"
      :count="pagination.total"
    />

    <!-- Filters -->
    <div class="mb-6 flex flex-col sm:flex-row gap-4">
      <!-- Search -->
      <div class="flex-1">
        <UInput
          v-model="search"
          icon="i-heroicons-magnifying-glass"
          placeholder="Search monsters..."
          size="lg"
        />
      </div>

      <!-- CR Filter -->
      <div class="w-full sm:w-48">
        <USelectMenu
          v-model="selectedCR"
          :options="crOptions"
          placeholder="Challenge Rating"
          size="lg"
        />
      </div>

      <!-- Type Filter -->
      <div class="w-full sm:w-48">
        <USelectMenu
          v-model="selectedType"
          :options="typeOptions"
          placeholder="Type"
          size="lg"
        />
      </div>
    </div>

    <!-- Results Count -->
    <UiListResultsCount
      :count="pagination.total"
      entity="monsters"
    />

    <!-- Loading State -->
    <div v-if="loading">
      <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 mb-8">
        <USkeleton
          v-for="i in 6"
          :key="i"
          class="h-64"
        />
      </div>
    </div>

    <!-- Error State -->
    <div v-else-if="error">
      <UAlert
        color="error"
        variant="soft"
        title="Failed to load monsters"
        description="There was an error loading the monsters list. Please try again later."
      />
    </div>

    <!-- Empty State -->
    <div
      v-else-if="!monsters || monsters.length === 0"
      class="text-center py-12"
    >
      <p class="text-gray-600 dark:text-gray-400">
        No monsters found matching your filters.
      </p>
    </div>

    <!-- Monster Grid -->
    <div
      v-else
      class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6 mb-8"
    >
      <MonsterCard
        v-for="monster in monsters"
        :key="monster.id"
        :monster="monster"
      />
    </div>

    <!-- Pagination -->
    <div
      v-if="pagination.total > pagination.per_page"
      class="flex justify-center mb-8"
    >
      <UPagination
        v-model:page="currentPage"
        :total="pagination.total"
        :items-per-page="pagination.per_page"
        show-edges
      />
    </div>

    <!-- Back to Top Link -->
    <div class="text-center">
      <UButton
        to="#"
        variant="ghost"
        color="neutral"
      >
        ↑ Back to Top
      </UButton>
    </div>

    <!-- Debug Panel -->
    <UiJsonDebugPanel
      :data="{ monsters, pagination, filters: { cr: selectedCR, type: selectedType } }"
    />
  </div>
</template>
```

**Step 2: Test list page in browser**

Run: `docker compose exec nuxt npm run dev`
Navigate to: `http://localhost:3000/monsters`

Expected:
- Page loads without errors
- Monsters display in grid
- Search works
- CR filter works
- Type filter works
- Pagination works

**Step 3: Test filters with query params**

Navigate to: `http://localhost:3000/monsters?cr=5-10&type=dragon`
Expected: Page loads with filters applied

**Step 4: Commit**

```bash
git add app/pages/monsters/index.vue
git commit -m "feat: Add monsters list page with CR/Type filters

- Grid layout with MonsterCard components
- Search by name/description
- Filter by CR range (Easy/Medium/Hard/Deadly)
- Filter by creature type (14 common types)
- Pagination support (15 per page)
- Loading/error/empty states
- SEO metadata
- Verified working in browser ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Create Monster Detail Page

**Files:**
- Create: `app/pages/monsters/[slug].vue`

**Step 1: Create monster detail page**

Create `app/pages/monsters/[slug].vue`:

```vue
<script setup lang="ts">
import type { Monster } from '~/types/api/entities'
import { getChallengeRatingColor } from '~/utils/badgeColors'

const route = useRoute()

// Fetch monster data and setup SEO
const { data: monster, loading, error } = useEntityDetail<Monster>({
  slug: route.params.slug as string,
  endpoint: '/monsters',
  cacheKey: 'monster',
  seo: {
    titleTemplate: name => `${name} - D&D 5e Monster`,
    descriptionExtractor: (monster: unknown) => {
      const m = monster as { description?: string }
      return m.description?.substring(0, 160) || ''
    },
    fallbackTitle: 'Monster - D&D 5e Compendium'
  }
})

/**
 * Format speeds into readable text
 */
const speedText = computed(() => {
  if (!monster.value) return ''
  const speeds: string[] = []

  if (monster.value.speed_walk) speeds.push(`${monster.value.speed_walk} ft.`)
  if (monster.value.speed_fly) speeds.push(`fly ${monster.value.speed_fly} ft.${monster.value.can_hover ? ' (hover)' : ''}`)
  if (monster.value.speed_swim) speeds.push(`swim ${monster.value.speed_swim} ft.`)
  if (monster.value.speed_burrow) speeds.push(`burrow ${monster.value.speed_burrow} ft.`)
  if (monster.value.speed_climb) speeds.push(`climb ${monster.value.speed_climb} ft.`)

  return speeds.join(', ')
})

/**
 * Quick stats for card display
 */
const quickStats = computed(() => {
  if (!monster.value) return []

  return [
    { label: 'Size', value: monster.value.size.name },
    { label: 'Type', value: monster.value.type },
    { label: 'Alignment', value: monster.value.alignment },
    { label: 'Armor Class', value: monster.value.armor_type ? `${monster.value.armor_class} (${monster.value.armor_type})` : monster.value.armor_class },
    { label: 'Hit Points', value: `${monster.value.hit_points_average} (${monster.value.hit_dice})` },
    { label: 'Speed', value: speedText.value },
    { label: 'STR', value: monster.value.strength },
    { label: 'DEX', value: monster.value.dexterity },
    { label: 'CON', value: monster.value.constitution },
    { label: 'INT', value: monster.value.intelligence },
    { label: 'WIS', value: monster.value.wisdom },
    { label: 'CHA', value: monster.value.charisma },
    { label: 'Challenge Rating', value: `${monster.value.challenge_rating} (${monster.value.experience_points.toLocaleString()} XP)` }
  ]
})

/**
 * Regular actions (not legendary)
 */
const regularActions = computed(() => {
  if (!monster.value?.actions) return []
  return monster.value.actions.filter(a => a.action_type === 'action')
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <!-- Loading State -->
    <UiDetailPageLoading
      v-if="loading"
      entity-type="monster"
    />

    <!-- Error State -->
    <UiDetailPageError
      v-else-if="error"
      entity-type="Monster"
    />

    <!-- Monster Content -->
    <div
      v-else-if="monster"
      class="space-y-8"
    >
      <!-- Breadcrumb -->
      <UiBackLink
        to="/monsters"
        label="Back to Monsters"
      />

      <!-- Header -->
      <UiDetailPageHeader
        :title="monster.name"
        :badges="[
          { label: `CR ${monster.challenge_rating}`, color: getChallengeRatingColor(monster.challenge_rating), variant: 'subtle' as const, size: 'lg' as const },
          { label: monster.type, color: 'neutral' as const, variant: 'subtle' as const, size: 'lg' as const },
          ...(monster.legendary_actions && monster.legendary_actions.length > 0 ? [{ label: '⭐ Legendary', color: 'warning' as const, variant: 'soft' as const, size: 'sm' as const }] : [])
        ]"
      />

      <!-- Quick Stats -->
      <UiDetailQuickStatsCard :stats="quickStats" />

      <!-- Description -->
      <div
        v-if="monster.description"
        class="prose dark:prose-invert max-w-none"
      >
        <p class="whitespace-pre-line">{{ monster.description }}</p>
      </div>

      <!-- Traits -->
      <UiAccordionTraits
        v-if="monster.traits && monster.traits.length > 0"
        :traits="monster.traits"
      />

      <!-- Actions -->
      <UiAccordionActions
        v-if="regularActions.length > 0"
        :actions="regularActions"
        title="Actions"
      />

      <!-- Legendary Actions -->
      <UiAccordionActions
        v-if="monster.legendary_actions && monster.legendary_actions.length > 0"
        :actions="monster.legendary_actions"
        title="Legendary Actions"
        :show-cost="true"
      />

      <!-- Modifiers -->
      <UiModifiersDisplay
        v-if="monster.modifiers && monster.modifiers.length > 0"
        :modifiers="monster.modifiers"
      />

      <!-- Conditions -->
      <UiAccordionBulletList
        v-if="monster.conditions && monster.conditions.length > 0"
        :items="monster.conditions"
        title="Conditions"
      />

      <!-- Sources -->
      <UiSourceDisplay
        v-if="monster.sources && monster.sources.length > 0"
        :sources="monster.sources"
      />

      <!-- Back to Monsters -->
      <div class="text-center">
        <UButton
          to="/monsters"
          variant="soft"
          color="neutral"
        >
          ← Back to Monsters
        </UButton>
      </div>

      <!-- Debug Panel -->
      <UiJsonDebugPanel :data="monster" />
    </div>
  </div>
</template>
```

**Step 2: Test detail page in browser**

Navigate to: `http://localhost:3000/monsters/ancient-red-dragon`

Expected:
- Page loads without errors
- Monster name and badges display
- Quick stats card shows all stats
- Description displays with proper formatting
- Traits accordion expands/collapses
- Actions accordion expands/collapses
- Legendary Actions accordion shows (if present)
- Modifiers display (if present)
- Sources display
- Dark mode works

**Step 3: Test multiple monsters**

Navigate to:
- `http://localhost:3000/monsters/aarakocra` (low CR)
- `http://localhost:3000/monsters/goblin` (common creature)

Expected: All monsters display correctly with appropriate data

**Step 4: Commit**

```bash
git add app/pages/monsters/[slug].vue
git commit -m "feat: Add monster detail page with full stat block

- Complete stat block display (AC, HP, speeds, ability scores)
- Traits, Actions, Legendary Actions accordions
- Modifiers and conditions sections
- Source citations
- SEO metadata
- Verified working in browser ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 8: Add Navigation Link

**Files:**
- Modify: `app/layouts/default.vue` (or appropriate nav component)

**Step 1: Find navigation component**

Run: `find app -name "*nav*" -o -name "*layout*" | grep -v node_modules`

Expected: Find layout or navigation file

**Step 2: Add Monsters link to navigation**

Add link between Items and Races:

```vue
<NuxtLink to="/monsters">Monsters</NuxtLink>
```

**Step 3: Test navigation in browser**

Navigate to: `http://localhost:3000`

Expected:
- Monsters link appears in navigation
- Clicking link goes to `/monsters`
- Link highlights when on monsters pages

**Step 4: Commit**

```bash
git add app/layouts/default.vue  # Or whichever file was modified
git commit -m "feat: Add Monsters link to main navigation

- Positioned between Items and Races
- Links to /monsters list page
- Verified navigation works ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 9: Final Verification & Documentation

**Files:**
- Modify: `CHANGELOG.md`
- Modify: `docs/CURRENT_STATUS.md`

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm test 2>&1 | tee /tmp/test-output.txt && tail -n 20 /tmp/test-output.txt`

Expected: All tests pass (~605-630 tests)

**Step 2: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck 2>&1 | grep -c "error TS"`

Expected: 13 errors or fewer (no regression)

**Step 3: Run linter**

Run: `docker compose exec nuxt npm run lint`

Expected: 0 errors

**Step 4: Test all pages in browser**

Test URLs:
- `http://localhost:3000/monsters` - List page
- `http://localhost:3000/monsters?cr=5-10` - CR filter
- `http://localhost:3000/monsters?type=dragon` - Type filter
- `http://localhost:3000/monsters?search=red` - Search
- `http://localhost:3000/monsters/ancient-red-dragon` - Detail page
- `http://localhost:3000/monsters/goblin` - Low CR detail

Expected: All pages work, no console errors

**Step 5: Test dark mode**

Toggle dark mode on all pages.

Expected: Dark mode works correctly

**Step 6: Test mobile responsiveness**

Resize browser to 375px, 768px, 1440px.

Expected: Layout responsive at all sizes

**Step 7: Update CHANGELOG.md**

Add to `[Unreleased]` section:

```markdown
### Added
- Monsters list page with CR/Type filtering (2025-11-22)
- Monster detail page with full stat block display (2025-11-22)
- MonsterCard component for list page (2025-11-22)
- UiAccordionActions component for actions/legendary actions (2025-11-22)
- UiAccordionTraits component for passive traits (2025-11-22)
- getChallengeRatingColor utility for CR badge colors (2025-11-22)
```

**Step 8: Update docs/CURRENT_STATUS.md**

Change line 6 from:
```
**6 of 6 Entity Types + 10 Reference Pages** (All Complete!)
```

To:
```
**7 of 7 Entity Types + 10 Reference Pages** (All Complete!)
```

Update entity list to include Monsters:
```
**✅ Spells, ✅ Items, ✅ Races, ✅ Classes, ✅ Backgrounds, ✅ Feats, ✅ Monsters**
```

Add monsters details:
```
- **Monsters:** CR/Type filters, legendary indicator, full stat blocks, traits/actions/legendary actions
```

**Step 9: Final commit**

```bash
git add CHANGELOG.md docs/CURRENT_STATUS.md
git commit -m "docs: Update CHANGELOG and status for Monsters feature

- Added 7th entity type (Monsters) to project
- 60+ new tests (all passing)
- TypeScript: 13 errors (no regression)
- ESLint: 0 errors

Monsters Feature Complete:
- List page with CR/Type filtering ✅
- Detail page with full stat blocks ✅
- 3 new components (MonsterCard, UiAccordionActions, UiAccordionTraits) ✅
- Reusable accordion components ✅
- Navigation link added ✅

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Success Checklist

Verify all criteria met:

- [ ] Monster type exported from OpenAPI types
- [ ] getChallengeRatingColor utility function added (10 tests)
- [ ] UiAccordionTraits component created (7 tests)
- [ ] UiAccordionActions component created (9 tests)
- [ ] MonsterCard component created (15+ tests)
- [ ] Monsters list page created and working
- [ ] Monster detail page created and working
- [ ] Navigation link added
- [ ] All 60+ new tests passing (total ~605-630)
- [ ] TypeScript: 13 errors or fewer
- [ ] ESLint: 0 errors
- [ ] Dark mode works on all pages
- [ ] Mobile responsive (375px - 1440px)
- [ ] CHANGELOG.md updated
- [ ] docs/CURRENT_STATUS.md updated
- [ ] All work committed with descriptive messages

---

## Implementation Notes

**TDD Workflow:**
- Every component follows RED-GREEN-REFACTOR
- Tests written before implementation
- Tests verified to fail before writing code
- Code written minimally to pass tests
- Refactor while keeping tests green

**Component Reuse:**
- MonsterCard reuses test helpers (hover, border, link, source)
- Accordion components follow existing patterns
- Badge color utility extends existing badge system
- Pages use existing composables (useEntityDetail, useEntityList)

**Estimated Time:**
- Task 1: 5 minutes
- Task 2: 15 minutes
- Task 3: 20 minutes
- Task 4: 20 minutes
- Task 5: 30 minutes
- Task 6: 20 minutes
- Task 7: 25 minutes
- Task 8: 5 minutes
- Task 9: 15 minutes
- **Total: ~2.5 hours**

**Reference Documents:**
- Design: `docs/plans/2025-11-22-monsters-feature-design.md`
- Current Status: `docs/CURRENT_STATUS.md`
- Existing Patterns: `app/pages/spells/`, `app/pages/items/`

---

**Plan Complete:** 2025-11-22
**Ready for Execution**
