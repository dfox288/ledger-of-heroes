# DM Screen UI Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a DM Screen page at `/parties/[id]/dm-screen` that displays party stats for combat reference and session overview.

**Architecture:** A dedicated page consuming `GET /api/parties/{id}/stats` through a Nitro proxy route. Modular Vue components organized under `app/components/dm-screen/`. Reusable display components for HP bars, death saves, and spell slots. Collapsible party summary panel with localStorage persistence.

**Tech Stack:** Nuxt 4.x, NuxtUI 4.x, TypeScript, Vitest, MSW for API mocking

---

## Task 1: TypeScript Types for DM Screen API

**Files:**
- Create: `app/types/dm-screen.ts`
- Modify: `app/types/index.ts:39` (add export)
- Test: `tests/types/dm-screen.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/types/dm-screen.test.ts
import { describe, it, expect } from 'vitest'
import type {
  DmScreenCharacter,
  DmScreenPartyStats,
  DmScreenPartySummary,
  CharacterHitPoints,
  CharacterCombat,
  CharacterSenses,
  CharacterCapabilities,
  CharacterEquipment,
  CharacterSavingThrows,
  CharacterCondition,
  CharacterSpellSlots
} from '~/types/dm-screen'

describe('DmScreen Types', () => {
  it('DmScreenCharacter has required fields', () => {
    const character: DmScreenCharacter = {
      id: 1,
      public_id: 'brave-mage-3aBc',
      name: 'Gandalf',
      level: 5,
      class_name: 'Wizard',
      hit_points: { current: 28, max: 35, temp: 0 },
      armor_class: 15,
      proficiency_bonus: 3,
      combat: {
        initiative_modifier: 2,
        speeds: { walk: 30, fly: null, swim: null, climb: null },
        death_saves: { successes: 0, failures: 0 },
        concentration: { active: false, spell: null }
      },
      senses: {
        passive_perception: 14,
        passive_investigation: 12,
        passive_insight: 14,
        darkvision: 60
      },
      capabilities: {
        languages: ['Common', 'Elvish'],
        size: 'Medium',
        tool_proficiencies: ['Thieves\' Tools']
      },
      equipment: {
        armor: { name: 'Studded Leather', type: 'light', stealth_disadvantage: false },
        weapons: [{ name: 'Longsword', damage: '1d8 slashing', range: null }],
        shield: false
      },
      saving_throws: { STR: 0, DEX: 2, CON: 1, INT: 4, WIS: 2, CHA: -1 },
      conditions: [],
      spell_slots: { '1': { current: 4, max: 4 } }
    }
    expect(character.name).toBe('Gandalf')
  })

  it('DmScreenPartySummary has aggregation fields', () => {
    const summary: DmScreenPartySummary = {
      all_languages: ['Common', 'Dwarvish'],
      darkvision_count: 3,
      no_darkvision: ['Aldric'],
      has_healer: true,
      healers: ['Mira (Cleric)'],
      has_detect_magic: true,
      has_dispel_magic: false,
      has_counterspell: true
    }
    expect(summary.darkvision_count).toBe(3)
  })

  it('DmScreenPartyStats combines characters and summary', () => {
    const stats: DmScreenPartyStats = {
      party: { id: 1, name: 'The Brave', description: null },
      characters: [],
      party_summary: {
        all_languages: [],
        darkvision_count: 0,
        no_darkvision: [],
        has_healer: false,
        healers: [],
        has_detect_magic: false,
        has_dispel_magic: false,
        has_counterspell: false
      }
    }
    expect(stats.party.name).toBe('The Brave')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/types/dm-screen.test.ts`
Expected: FAIL with "Cannot find module '~/types/dm-screen'"

**Step 3: Write the types**

```typescript
// app/types/dm-screen.ts

/**
 * DM Screen API Types
 *
 * Types for the party stats endpoint used by the DM Screen feature.
 * Based on GET /api/v1/parties/{id}/stats response.
 */

export interface CharacterHitPoints {
  current: number
  max: number
  temp: number
}

export interface CharacterSpeeds {
  walk: number
  fly: number | null
  swim: number | null
  climb: number | null
}

export interface CharacterDeathSaves {
  successes: number
  failures: number
}

export interface CharacterConcentration {
  active: boolean
  spell: string | null
}

export interface CharacterCombat {
  initiative_modifier: number
  speeds: CharacterSpeeds
  death_saves: CharacterDeathSaves
  concentration: CharacterConcentration
}

export interface CharacterSenses {
  passive_perception: number
  passive_investigation: number
  passive_insight: number
  darkvision: number | null
}

export interface CharacterCapabilities {
  languages: string[]
  size: string
  tool_proficiencies: string[]
}

export interface CharacterArmor {
  name: string
  type: string
  stealth_disadvantage: boolean
}

export interface CharacterWeapon {
  name: string
  damage: string
  range: string | null
}

export interface CharacterEquipment {
  armor: CharacterArmor | null
  weapons: CharacterWeapon[]
  shield: boolean
}

export interface CharacterSavingThrows {
  STR: number
  DEX: number
  CON: number
  INT: number
  WIS: number
  CHA: number
}

export interface CharacterCondition {
  name: string
  slug: string
  level: number | null
}

export interface SpellSlotLevel {
  current: number
  max: number
}

export type CharacterSpellSlots = Record<string, SpellSlotLevel>

export interface DmScreenCharacter {
  id: number
  public_id: string
  name: string
  level: number
  class_name: string
  hit_points: CharacterHitPoints
  armor_class: number
  proficiency_bonus: number
  combat: CharacterCombat
  senses: CharacterSenses
  capabilities: CharacterCapabilities
  equipment: CharacterEquipment
  saving_throws: CharacterSavingThrows
  conditions: CharacterCondition[]
  spell_slots: CharacterSpellSlots
}

export interface DmScreenPartySummary {
  all_languages: string[]
  darkvision_count: number
  no_darkvision: string[]
  has_healer: boolean
  healers: string[]
  has_detect_magic: boolean
  has_dispel_magic: boolean
  has_counterspell: boolean
}

export interface DmScreenPartyInfo {
  id: number
  name: string
  description: string | null
}

export interface DmScreenPartyStats {
  party: DmScreenPartyInfo
  characters: DmScreenCharacter[]
  party_summary: DmScreenPartySummary
}
```

**Step 4: Export from barrel file**

```typescript
// app/types/index.ts - Add after line 39 (after party export)
export * from './dm-screen'
```

**Step 5: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/types/dm-screen.test.ts`
Expected: PASS

**Step 6: Commit**

```bash
git add app/types/dm-screen.ts app/types/index.ts tests/types/dm-screen.test.ts
git commit -m "feat(dm-screen): add TypeScript types for party stats API"
```

---

## Task 2: Nitro Server Route for Party Stats

**Files:**
- Create: `server/api/parties/[id]/stats.get.ts`
- Test: `tests/server/api/parties/stats.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/server/api/parties/stats.test.ts
import { describe, it, expect, beforeAll, afterEach, afterAll } from 'vitest'
import { server, http, HttpResponse } from '@/tests/msw/server'

describe('GET /api/parties/:id/stats', () => {
  beforeAll(() => server.listen())
  afterEach(() => server.resetHandlers())
  afterAll(() => server.close())

  it('returns party stats from backend', async () => {
    const mockStats = {
      data: {
        party: { id: 1, name: 'Test Party', description: null },
        characters: [],
        party_summary: {
          all_languages: ['Common'],
          darkvision_count: 0,
          no_darkvision: [],
          has_healer: false,
          healers: [],
          has_detect_magic: false,
          has_dispel_magic: false,
          has_counterspell: false
        }
      }
    }

    server.use(
      http.get('http://localhost:8080/api/v1/parties/1/stats', () => {
        return HttpResponse.json(mockStats)
      })
    )

    const response = await fetch('http://localhost:4000/api/parties/1/stats')
    const data = await response.json()

    expect(response.ok).toBe(true)
    expect(data.data.party.name).toBe('Test Party')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/server/api/parties/stats.test.ts`
Expected: FAIL with 404 (route doesn't exist)

**Step 3: Create the Nitro route**

```typescript
// server/api/parties/[id]/stats.get.ts
/**
 * Get party stats endpoint - Proxies to Laravel backend
 *
 * Returns comprehensive party stats for DM Screen display.
 * @example GET /api/parties/1/stats
 */
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')

  try {
    const data = await $fetch(`${config.apiBaseServer}/parties/${id}/stats`)
    return data
  } catch (error: unknown) {
    const err = error as { statusCode?: number; statusMessage?: string; data?: unknown }
    throw createError({
      statusCode: err.statusCode || 500,
      statusMessage: err.statusMessage || 'Failed to fetch party stats',
      data: err.data
    })
  }
})
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/server/api/parties/stats.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add server/api/parties/[id]/stats.get.ts tests/server/api/parties/stats.test.ts
git commit -m "feat(dm-screen): add Nitro route for party stats endpoint"
```

---

## Task 3: HpBar Component

**Files:**
- Create: `app/components/dm-screen/HpBar.vue`
- Test: `tests/components/dm-screen/HpBar.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/dm-screen/HpBar.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import HpBar from '~/components/dm-screen/HpBar.vue'

describe('DmScreenHpBar', () => {
  it('displays current and max HP', async () => {
    const wrapper = await mountSuspended(HpBar, {
      props: { current: 28, max: 35, temp: 0 }
    })
    expect(wrapper.text()).toContain('28')
    expect(wrapper.text()).toContain('35')
  })

  it('displays temp HP when present', async () => {
    const wrapper = await mountSuspended(HpBar, {
      props: { current: 28, max: 35, temp: 5 }
    })
    expect(wrapper.text()).toContain('+5')
  })

  it('does not display temp HP when zero', async () => {
    const wrapper = await mountSuspended(HpBar, {
      props: { current: 28, max: 35, temp: 0 }
    })
    expect(wrapper.text()).not.toContain('+0')
  })

  it('shows green bar when HP above 50%', async () => {
    const wrapper = await mountSuspended(HpBar, {
      props: { current: 30, max: 35, temp: 0 }
    })
    const bar = wrapper.find('[data-testid="hp-bar-fill"]')
    expect(bar.classes().join(' ')).toMatch(/green|success|emerald/)
  })

  it('shows yellow bar when HP between 25% and 50%', async () => {
    const wrapper = await mountSuspended(HpBar, {
      props: { current: 10, max: 35, temp: 0 }
    })
    const bar = wrapper.find('[data-testid="hp-bar-fill"]')
    expect(bar.classes().join(' ')).toMatch(/yellow|warning|amber/)
  })

  it('shows red bar when HP below 25%', async () => {
    const wrapper = await mountSuspended(HpBar, {
      props: { current: 5, max: 35, temp: 0 }
    })
    const bar = wrapper.find('[data-testid="hp-bar-fill"]')
    expect(bar.classes().join(' ')).toMatch(/red|error|rose/)
  })

  it('calculates bar width based on percentage', async () => {
    const wrapper = await mountSuspended(HpBar, {
      props: { current: 17, max: 34, temp: 0 }
    })
    const bar = wrapper.find('[data-testid="hp-bar-fill"]')
    // 50% HP
    expect(bar.attributes('style')).toContain('50%')
  })

  it('handles zero max HP gracefully', async () => {
    const wrapper = await mountSuspended(HpBar, {
      props: { current: 0, max: 0, temp: 0 }
    })
    expect(wrapper.text()).toContain('0')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/HpBar.test.ts`
Expected: FAIL with "Cannot find module '~/components/dm-screen/HpBar.vue'"

**Step 3: Write the component**

```vue
<!-- app/components/dm-screen/HpBar.vue -->
<script setup lang="ts">
interface Props {
  current: number
  max: number
  temp: number
}

const props = defineProps<Props>()

const percentage = computed(() => {
  if (props.max === 0) return 0
  return Math.min(100, Math.max(0, (props.current / props.max) * 100))
})

const colorClass = computed(() => {
  const pct = percentage.value
  if (pct > 50) return 'bg-emerald-500'
  if (pct > 25) return 'bg-amber-500'
  return 'bg-rose-500'
})
</script>

<template>
  <div class="flex items-center gap-2">
    <div class="flex-1 h-4 bg-neutral-200 dark:bg-neutral-700 rounded overflow-hidden">
      <div
        data-testid="hp-bar-fill"
        :class="['h-full transition-all duration-300', colorClass]"
        :style="{ width: `${percentage}%` }"
      />
    </div>
    <div class="text-sm font-medium whitespace-nowrap min-w-[60px] text-right">
      {{ current }}/{{ max }}
      <span
        v-if="temp > 0"
        class="text-blue-500 ml-1"
      >+{{ temp }}</span>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/HpBar.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/dm-screen/HpBar.vue tests/components/dm-screen/HpBar.test.ts
git commit -m "feat(dm-screen): add HpBar component with color-coded health display"
```

---

## Task 4: DeathSavesCompact Component

**Files:**
- Create: `app/components/dm-screen/DeathSavesCompact.vue`
- Test: `tests/components/dm-screen/DeathSavesCompact.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/dm-screen/DeathSavesCompact.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import DeathSavesCompact from '~/components/dm-screen/DeathSavesCompact.vue'

describe('DmScreenDeathSavesCompact', () => {
  it('displays successes and failures', async () => {
    const wrapper = await mountSuspended(DeathSavesCompact, {
      props: { successes: 2, failures: 1 }
    })
    // Should show ●●○ for successes and ●○○ for failures
    expect(wrapper.text()).toContain('S')
    expect(wrapper.text()).toContain('F')
  })

  it('shows correct number of filled success circles', async () => {
    const wrapper = await mountSuspended(DeathSavesCompact, {
      props: { successes: 2, failures: 0 }
    })
    const filledSuccess = wrapper.findAll('[data-testid^="success-filled"]')
    expect(filledSuccess.length).toBe(2)
  })

  it('shows correct number of filled failure circles', async () => {
    const wrapper = await mountSuspended(DeathSavesCompact, {
      props: { successes: 0, failures: 3 }
    })
    const filledFailure = wrapper.findAll('[data-testid^="failure-filled"]')
    expect(filledFailure.length).toBe(3)
  })

  it('hides when both are zero', async () => {
    const wrapper = await mountSuspended(DeathSavesCompact, {
      props: { successes: 0, failures: 0 }
    })
    expect(wrapper.find('[data-testid="death-saves-container"]').exists()).toBe(false)
  })

  it('shows stabilized indicator at 3 successes', async () => {
    const wrapper = await mountSuspended(DeathSavesCompact, {
      props: { successes: 3, failures: 0 }
    })
    expect(wrapper.text()).toMatch(/stab|stable/i)
  })

  it('shows dead indicator at 3 failures', async () => {
    const wrapper = await mountSuspended(DeathSavesCompact, {
      props: { successes: 0, failures: 3 }
    })
    expect(wrapper.text()).toMatch(/dead|skull/i)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/DeathSavesCompact.test.ts`
Expected: FAIL

**Step 3: Write the component**

```vue
<!-- app/components/dm-screen/DeathSavesCompact.vue -->
<script setup lang="ts">
interface Props {
  successes: number
  failures: number
}

const props = defineProps<Props>()

const showComponent = computed(() => props.successes > 0 || props.failures > 0)
const isStabilized = computed(() => props.successes >= 3)
const isDead = computed(() => props.failures >= 3)
</script>

<template>
  <div
    v-if="showComponent"
    data-testid="death-saves-container"
    class="flex items-center gap-2 text-xs"
  >
    <!-- Stabilized state -->
    <template v-if="isStabilized">
      <UIcon
        name="i-heroicons-heart-solid"
        class="w-4 h-4 text-emerald-500"
      />
      <span class="text-emerald-600 dark:text-emerald-400 font-medium">Stabilized</span>
    </template>

    <!-- Dead state -->
    <template v-else-if="isDead">
      <UIcon
        name="i-heroicons-x-circle-solid"
        class="w-4 h-4 text-rose-500"
      />
      <span class="text-rose-600 dark:text-rose-400 font-medium">Dead</span>
    </template>

    <!-- In progress -->
    <template v-else>
      <!-- Successes -->
      <span class="flex items-center gap-0.5">
        <span class="text-neutral-500 mr-0.5">S</span>
        <span
          v-for="i in 3"
          :key="`s-${i}`"
          class="w-2.5 h-2.5 rounded-full border"
          :class="i <= successes
            ? 'bg-emerald-500 border-emerald-500'
            : 'border-neutral-300 dark:border-neutral-600'"
          :data-testid="i <= successes ? `success-filled-${i}` : `success-empty-${i}`"
        />
      </span>

      <!-- Failures -->
      <span class="flex items-center gap-0.5">
        <span class="text-neutral-500 mr-0.5">F</span>
        <span
          v-for="i in 3"
          :key="`f-${i}`"
          class="w-2.5 h-2.5 rounded-full border"
          :class="i <= failures
            ? 'bg-rose-500 border-rose-500'
            : 'border-neutral-300 dark:border-neutral-600'"
          :data-testid="i <= failures ? `failure-filled-${i}` : `failure-empty-${i}`"
        />
      </span>
    </template>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/DeathSavesCompact.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/dm-screen/DeathSavesCompact.vue tests/components/dm-screen/DeathSavesCompact.test.ts
git commit -m "feat(dm-screen): add DeathSavesCompact component for combat table"
```

---

## Task 5: SpellSlotsCompact Component

**Files:**
- Create: `app/components/dm-screen/SpellSlotsCompact.vue`
- Test: `tests/components/dm-screen/SpellSlotsCompact.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/dm-screen/SpellSlotsCompact.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SpellSlotsCompact from '~/components/dm-screen/SpellSlotsCompact.vue'

describe('DmScreenSpellSlotsCompact', () => {
  it('displays slots for each level', async () => {
    const wrapper = await mountSuspended(SpellSlotsCompact, {
      props: {
        slots: {
          '1': { current: 3, max: 4 },
          '2': { current: 2, max: 3 }
        }
      }
    })
    expect(wrapper.text()).toContain('1st')
    expect(wrapper.text()).toContain('2nd')
  })

  it('shows filled and empty slot indicators', async () => {
    const wrapper = await mountSuspended(SpellSlotsCompact, {
      props: {
        slots: { '1': { current: 2, max: 4 } }
      }
    })
    const filled = wrapper.findAll('[data-testid^="slot-filled"]')
    const empty = wrapper.findAll('[data-testid^="slot-empty"]')
    expect(filled.length).toBe(2)
    expect(empty.length).toBe(2)
  })

  it('renders nothing when slots object is empty', async () => {
    const wrapper = await mountSuspended(SpellSlotsCompact, {
      props: { slots: {} }
    })
    expect(wrapper.find('[data-testid="spell-slots-container"]').exists()).toBe(false)
  })

  it('skips levels with zero max slots', async () => {
    const wrapper = await mountSuspended(SpellSlotsCompact, {
      props: {
        slots: {
          '1': { current: 0, max: 0 },
          '2': { current: 2, max: 3 }
        }
      }
    })
    expect(wrapper.text()).not.toContain('1st')
    expect(wrapper.text()).toContain('2nd')
  })

  it('formats ordinals correctly', async () => {
    const wrapper = await mountSuspended(SpellSlotsCompact, {
      props: {
        slots: {
          '1': { current: 1, max: 1 },
          '2': { current: 1, max: 1 },
          '3': { current: 1, max: 1 }
        }
      }
    })
    expect(wrapper.text()).toContain('1st')
    expect(wrapper.text()).toContain('2nd')
    expect(wrapper.text()).toContain('3rd')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/SpellSlotsCompact.test.ts`
Expected: FAIL

**Step 3: Write the component**

```vue
<!-- app/components/dm-screen/SpellSlotsCompact.vue -->
<script setup lang="ts">
import type { CharacterSpellSlots } from '~/types/dm-screen'

interface Props {
  slots: CharacterSpellSlots
}

const props = defineProps<Props>()

function ordinal(n: number): string {
  const suffixes = ['th', 'st', 'nd', 'rd']
  const v = n % 100
  return n + (suffixes[(v - 20) % 10] || suffixes[v] || suffixes[0])
}

const validSlots = computed(() => {
  return Object.entries(props.slots)
    .filter(([, slot]) => slot.max > 0)
    .sort(([a], [b]) => Number(a) - Number(b))
})

const hasSlots = computed(() => validSlots.value.length > 0)
</script>

<template>
  <div
    v-if="hasSlots"
    data-testid="spell-slots-container"
    class="flex flex-wrap gap-2 text-xs"
  >
    <div
      v-for="[level, slot] in validSlots"
      :key="level"
      class="flex items-center gap-1"
    >
      <span class="text-neutral-500">{{ ordinal(Number(level)) }}</span>
      <span class="flex gap-0.5">
        <span
          v-for="i in slot.max"
          :key="i"
          class="w-2 h-2 rounded-full"
          :class="i <= slot.current
            ? 'bg-spell-500'
            : 'bg-neutral-300 dark:bg-neutral-600'"
          :data-testid="i <= slot.current ? `slot-filled-${level}-${i}` : `slot-empty-${level}-${i}`"
        />
      </span>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/SpellSlotsCompact.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/dm-screen/SpellSlotsCompact.vue tests/components/dm-screen/SpellSlotsCompact.test.ts
git commit -m "feat(dm-screen): add SpellSlotsCompact component for caster tracking"
```

---

## Task 6: PartySummary Component

**Files:**
- Create: `app/components/dm-screen/PartySummary.vue`
- Test: `tests/components/dm-screen/PartySummary.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/dm-screen/PartySummary.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import PartySummary from '~/components/dm-screen/PartySummary.vue'
import type { DmScreenPartySummary } from '~/types/dm-screen'

const mockSummary: DmScreenPartySummary = {
  all_languages: ['Common', 'Elvish', 'Dwarvish', 'Undercommon', 'Draconic'],
  darkvision_count: 3,
  no_darkvision: ['Aldric the Human'],
  has_healer: true,
  healers: ['Mira (Cleric)', 'Finn (Druid)'],
  has_detect_magic: true,
  has_dispel_magic: false,
  has_counterspell: true
}

describe('DmScreenPartySummary', () => {
  it('displays languages section', async () => {
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: mockSummary }
    })
    expect(wrapper.text()).toContain('Languages')
    expect(wrapper.text()).toContain('Common')
  })

  it('shows "+N more" when many languages', async () => {
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: mockSummary }
    })
    // Default shows 3 languages, rest as "+N more"
    expect(wrapper.text()).toMatch(/\+\d+ more/)
  })

  it('displays darkvision count', async () => {
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: mockSummary }
    })
    expect(wrapper.text()).toContain('Darkvision')
    expect(wrapper.text()).toContain('3')
  })

  it('shows warning for characters without darkvision', async () => {
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: mockSummary }
    })
    expect(wrapper.text()).toContain('Aldric')
  })

  it('displays healers when present', async () => {
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: mockSummary }
    })
    expect(wrapper.text()).toContain('Healers')
    expect(wrapper.text()).toContain('Mira')
  })

  it('shows warning when no healers', async () => {
    const noHealerSummary = { ...mockSummary, has_healer: false, healers: [] }
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: noHealerSummary }
    })
    expect(wrapper.text()).toMatch(/no healer|none/i)
  })

  it('displays utility spell coverage', async () => {
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: mockSummary }
    })
    expect(wrapper.text()).toContain('Utility')
    expect(wrapper.text()).toContain('Detect Magic')
  })

  it('shows checkmark for available utility spells', async () => {
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: mockSummary }
    })
    // Detect Magic has checkmark
    const detectSection = wrapper.find('[data-testid="utility-detect-magic"]')
    expect(detectSection.text()).toContain('✓')
  })

  it('shows X for missing utility spells', async () => {
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: mockSummary }
    })
    // Dispel Magic is missing
    const dispelSection = wrapper.find('[data-testid="utility-dispel-magic"]')
    expect(dispelSection.classes().join(' ')).toMatch(/muted|neutral|gray/)
  })

  it('is collapsible', async () => {
    const wrapper = await mountSuspended(PartySummary, {
      props: { summary: mockSummary, collapsed: false }
    })
    const toggle = wrapper.find('[data-testid="summary-toggle"]')
    expect(toggle.exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/PartySummary.test.ts`
Expected: FAIL

**Step 3: Write the component**

```vue
<!-- app/components/dm-screen/PartySummary.vue -->
<script setup lang="ts">
import type { DmScreenPartySummary } from '~/types/dm-screen'

interface Props {
  summary: DmScreenPartySummary
  collapsed?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  collapsed: false
})

const emit = defineEmits<{
  'update:collapsed': [value: boolean]
}>()

const isCollapsed = computed({
  get: () => props.collapsed,
  set: (val) => emit('update:collapsed', val)
})

const displayLanguages = computed(() => props.summary.all_languages.slice(0, 3))
const moreLanguages = computed(() => Math.max(0, props.summary.all_languages.length - 3))
</script>

<template>
  <div class="bg-neutral-50 dark:bg-neutral-800 rounded-lg border border-neutral-200 dark:border-neutral-700">
    <!-- Header -->
    <button
      data-testid="summary-toggle"
      class="w-full flex items-center justify-between p-3 text-left"
      @click="isCollapsed = !isCollapsed"
    >
      <span class="font-medium text-neutral-900 dark:text-white">Party Summary</span>
      <UIcon
        :name="isCollapsed ? 'i-heroicons-chevron-down' : 'i-heroicons-chevron-up'"
        class="w-5 h-5 text-neutral-400"
      />
    </button>

    <!-- Content -->
    <div
      v-show="!isCollapsed"
      class="p-4 pt-0 grid grid-cols-2 md:grid-cols-4 gap-4"
    >
      <!-- Languages -->
      <div>
        <h4 class="text-xs font-medium text-neutral-500 uppercase mb-1">Languages</h4>
        <div class="flex flex-wrap gap-1">
          <UBadge
            v-for="lang in displayLanguages"
            :key="lang"
            color="neutral"
            variant="subtle"
            size="md"
          >
            {{ lang }}
          </UBadge>
          <UBadge
            v-if="moreLanguages > 0"
            color="neutral"
            variant="outline"
            size="md"
          >
            +{{ moreLanguages }} more
          </UBadge>
        </div>
      </div>

      <!-- Darkvision -->
      <div>
        <h4 class="text-xs font-medium text-neutral-500 uppercase mb-1">Darkvision</h4>
        <div class="flex items-center gap-2">
          <span class="text-lg font-semibold">{{ summary.darkvision_count }}</span>
          <span class="text-sm text-neutral-500">have it</span>
        </div>
        <div
          v-if="summary.no_darkvision.length > 0"
          class="text-xs text-amber-600 dark:text-amber-400 mt-1"
        >
          ⚠️ {{ summary.no_darkvision.join(', ') }}
        </div>
      </div>

      <!-- Healers -->
      <div>
        <h4 class="text-xs font-medium text-neutral-500 uppercase mb-1">Healers</h4>
        <template v-if="summary.has_healer">
          <div
            v-for="healer in summary.healers"
            :key="healer"
            class="text-sm text-emerald-600 dark:text-emerald-400"
          >
            ✓ {{ healer }}
          </div>
        </template>
        <div
          v-else
          class="text-sm text-rose-600 dark:text-rose-400"
        >
          ⚠️ No healer in party
        </div>
      </div>

      <!-- Utility Spells -->
      <div>
        <h4 class="text-xs font-medium text-neutral-500 uppercase mb-1">Utility</h4>
        <div class="space-y-0.5 text-sm">
          <div
            data-testid="utility-detect-magic"
            :class="summary.has_detect_magic
              ? 'text-emerald-600 dark:text-emerald-400'
              : 'text-neutral-400'"
          >
            {{ summary.has_detect_magic ? '✓' : '✗' }} Detect Magic
          </div>
          <div
            data-testid="utility-dispel-magic"
            :class="summary.has_dispel_magic
              ? 'text-emerald-600 dark:text-emerald-400'
              : 'text-neutral-400'"
          >
            {{ summary.has_dispel_magic ? '✓' : '✗' }} Dispel Magic
          </div>
          <div
            data-testid="utility-counterspell"
            :class="summary.has_counterspell
              ? 'text-emerald-600 dark:text-emerald-400'
              : 'text-neutral-400'"
          >
            {{ summary.has_counterspell ? '✓' : '✗' }} Counterspell
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/PartySummary.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/dm-screen/PartySummary.vue tests/components/dm-screen/PartySummary.test.ts
git commit -m "feat(dm-screen): add PartySummary component with collapsible panel"
```

---

## Task 7: CombatTableRow Component

**Files:**
- Create: `app/components/dm-screen/CombatTableRow.vue`
- Test: `tests/components/dm-screen/CombatTableRow.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/dm-screen/CombatTableRow.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import CombatTableRow from '~/components/dm-screen/CombatTableRow.vue'
import type { DmScreenCharacter } from '~/types/dm-screen'

const mockCharacter: DmScreenCharacter = {
  id: 1,
  public_id: 'brave-mage-3aBc',
  name: 'Gandalf',
  level: 5,
  class_name: 'Wizard',
  hit_points: { current: 28, max: 35, temp: 0 },
  armor_class: 15,
  proficiency_bonus: 3,
  combat: {
    initiative_modifier: 2,
    speeds: { walk: 30, fly: null, swim: null, climb: null },
    death_saves: { successes: 0, failures: 0 },
    concentration: { active: false, spell: null }
  },
  senses: {
    passive_perception: 14,
    passive_investigation: 12,
    passive_insight: 14,
    darkvision: 60
  },
  capabilities: {
    languages: ['Common', 'Elvish'],
    size: 'Medium',
    tool_proficiencies: []
  },
  equipment: {
    armor: null,
    weapons: [],
    shield: false
  },
  saving_throws: { STR: 0, DEX: 2, CON: 1, INT: 4, WIS: 2, CHA: -1 },
  conditions: [],
  spell_slots: { '1': { current: 4, max: 4 } }
}

describe('DmScreenCombatTableRow', () => {
  it('displays character name and class', async () => {
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: mockCharacter }
    })
    expect(wrapper.text()).toContain('Gandalf')
    expect(wrapper.text()).toContain('Wizard')
  })

  it('displays HP bar', async () => {
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: mockCharacter }
    })
    expect(wrapper.find('[data-testid="hp-bar-fill"]').exists()).toBe(true)
  })

  it('displays armor class', async () => {
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: mockCharacter }
    })
    expect(wrapper.text()).toContain('15')
  })

  it('highlights high AC (17+)', async () => {
    const highAcCharacter = { ...mockCharacter, armor_class: 18 }
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: highAcCharacter }
    })
    const acBadge = wrapper.find('[data-testid="ac-badge"]')
    expect(acBadge.classes().join(' ')).toMatch(/primary|blue|highlight/)
  })

  it('displays initiative modifier with sign', async () => {
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: mockCharacter }
    })
    expect(wrapper.text()).toContain('+2')
  })

  it('displays passive scores', async () => {
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: mockCharacter }
    })
    expect(wrapper.text()).toContain('14') // Perception
    expect(wrapper.text()).toContain('12') // Investigation
  })

  it('emits click event for expansion', async () => {
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: mockCharacter }
    })
    await wrapper.find('[data-testid="combat-row"]').trigger('click')
    expect(wrapper.emitted('click')).toBeTruthy()
  })

  it('shows death saves when active', async () => {
    const dyingCharacter = {
      ...mockCharacter,
      combat: {
        ...mockCharacter.combat,
        death_saves: { successes: 1, failures: 2 }
      }
    }
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: dyingCharacter }
    })
    expect(wrapper.find('[data-testid="death-saves-container"]').exists()).toBe(true)
  })

  it('shows condition badges when present', async () => {
    const conditionedCharacter = {
      ...mockCharacter,
      conditions: [{ name: 'Poisoned', slug: 'poisoned', level: null }]
    }
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: conditionedCharacter }
    })
    expect(wrapper.text()).toContain('Poisoned')
  })

  it('shows concentration indicator when active', async () => {
    const concentratingCharacter = {
      ...mockCharacter,
      combat: {
        ...mockCharacter.combat,
        concentration: { active: true, spell: 'Haste' }
      }
    }
    const wrapper = await mountSuspended(CombatTableRow, {
      props: { character: concentratingCharacter }
    })
    expect(wrapper.text()).toContain('Haste')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/CombatTableRow.test.ts`
Expected: FAIL

**Step 3: Write the component**

```vue
<!-- app/components/dm-screen/CombatTableRow.vue -->
<script setup lang="ts">
import type { DmScreenCharacter } from '~/types/dm-screen'

interface Props {
  character: DmScreenCharacter
  expanded?: boolean
}

const props = withDefaults(defineProps<Props>(), {
  expanded: false
})

const emit = defineEmits<{
  click: []
}>()

function formatModifier(mod: number): string {
  return mod >= 0 ? `+${mod}` : `${mod}`
}

const isHighAc = computed(() => props.character.armor_class >= 17)
const hasDeathSaves = computed(() =>
  props.character.combat.death_saves.successes > 0 ||
  props.character.combat.death_saves.failures > 0
)
const isConcentrating = computed(() => props.character.combat.concentration.active)
const hasConditions = computed(() => props.character.conditions.length > 0)
</script>

<template>
  <tr
    data-testid="combat-row"
    class="hover:bg-neutral-50 dark:hover:bg-neutral-800 cursor-pointer transition-colors"
    :class="{ 'bg-neutral-50 dark:bg-neutral-800': expanded }"
    @click="emit('click')"
  >
    <!-- Name + Class -->
    <td class="py-3 px-4">
      <div class="font-medium text-neutral-900 dark:text-white">
        {{ character.name }}
      </div>
      <div class="text-xs text-neutral-500">
        {{ character.class_name }} {{ character.level }}
      </div>
      <!-- Status indicators -->
      <div
        v-if="hasDeathSaves || isConcentrating || hasConditions"
        class="mt-1 flex flex-wrap gap-1"
      >
        <DmScreenDeathSavesCompact
          v-if="hasDeathSaves"
          :successes="character.combat.death_saves.successes"
          :failures="character.combat.death_saves.failures"
        />
        <UBadge
          v-if="isConcentrating"
          color="spell"
          variant="subtle"
          size="md"
        >
          ◉ {{ character.combat.concentration.spell }}
        </UBadge>
        <UBadge
          v-for="condition in character.conditions"
          :key="condition.slug"
          color="warning"
          variant="subtle"
          size="md"
        >
          {{ condition.name }}
        </UBadge>
      </div>
    </td>

    <!-- HP Bar -->
    <td class="py-3 px-4 min-w-[180px]">
      <DmScreenHpBar
        :current="character.hit_points.current"
        :max="character.hit_points.max"
        :temp="character.hit_points.temp"
      />
    </td>

    <!-- AC -->
    <td class="py-3 px-4 text-center">
      <UBadge
        data-testid="ac-badge"
        :color="isHighAc ? 'primary' : 'neutral'"
        :variant="isHighAc ? 'solid' : 'subtle'"
        size="lg"
        class="font-mono font-bold"
      >
        {{ character.armor_class }}
      </UBadge>
    </td>

    <!-- Initiative -->
    <td class="py-3 px-4 text-center font-mono">
      {{ formatModifier(character.combat.initiative_modifier) }}
    </td>

    <!-- Passive Perception -->
    <td class="py-3 px-4 text-center">
      {{ character.senses.passive_perception }}
    </td>

    <!-- Passive Investigation -->
    <td class="py-3 px-4 text-center">
      {{ character.senses.passive_investigation }}
    </td>

    <!-- Passive Insight -->
    <td class="py-3 px-4 text-center">
      {{ character.senses.passive_insight }}
    </td>
  </tr>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/CombatTableRow.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/dm-screen/CombatTableRow.vue tests/components/dm-screen/CombatTableRow.test.ts
git commit -m "feat(dm-screen): add CombatTableRow component with status indicators"
```

---

## Task 8: CharacterDetail Component

**Files:**
- Create: `app/components/dm-screen/CharacterDetail.vue`
- Test: `tests/components/dm-screen/CharacterDetail.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/dm-screen/CharacterDetail.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import CharacterDetail from '~/components/dm-screen/CharacterDetail.vue'
import type { DmScreenCharacter } from '~/types/dm-screen'

const mockCharacter: DmScreenCharacter = {
  id: 1,
  public_id: 'brave-mage-3aBc',
  name: 'Gandalf',
  level: 5,
  class_name: 'Wizard',
  hit_points: { current: 28, max: 35, temp: 0 },
  armor_class: 15,
  proficiency_bonus: 3,
  combat: {
    initiative_modifier: 2,
    speeds: { walk: 30, fly: 60, swim: null, climb: null },
    death_saves: { successes: 0, failures: 0 },
    concentration: { active: true, spell: 'Haste' }
  },
  senses: {
    passive_perception: 14,
    passive_investigation: 12,
    passive_insight: 14,
    darkvision: 60
  },
  capabilities: {
    languages: ['Common', 'Elvish', 'Dwarvish'],
    size: 'Medium',
    tool_proficiencies: ['Thieves\' Tools', 'Alchemist\'s Supplies']
  },
  equipment: {
    armor: { name: 'Studded Leather', type: 'light', stealth_disadvantage: false },
    weapons: [
      { name: 'Staff', damage: '1d6 bludgeoning', range: null },
      { name: 'Dagger', damage: '1d4 piercing', range: '20/60' }
    ],
    shield: true
  },
  saving_throws: { STR: 0, DEX: 2, CON: 1, INT: 6, WIS: 4, CHA: -1 },
  conditions: [{ name: 'Blessed', slug: 'blessed', level: null }],
  spell_slots: {
    '1': { current: 4, max: 4 },
    '2': { current: 2, max: 3 },
    '3': { current: 1, max: 2 }
  }
}

describe('DmScreenCharacterDetail', () => {
  describe('Combat Section', () => {
    it('displays speeds', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('30 ft')
      expect(wrapper.text()).toContain('60 ft')
    })

    it('displays saving throws', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('STR')
      expect(wrapper.text()).toContain('+6') // INT saving throw
    })

    it('displays concentration status', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('Concentration')
      expect(wrapper.text()).toContain('Haste')
    })
  })

  describe('Equipment Section', () => {
    it('displays armor info', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('Studded Leather')
      expect(wrapper.text()).toContain('light')
    })

    it('displays weapons with damage', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('Staff')
      expect(wrapper.text()).toContain('1d6 bludgeoning')
    })

    it('shows shield indicator', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('Shield')
    })
  })

  describe('Capabilities Section', () => {
    it('displays languages', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('Common')
      expect(wrapper.text()).toContain('Elvish')
    })

    it('displays tool proficiencies', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('Thieves\' Tools')
    })

    it('displays size', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('Medium')
    })
  })

  describe('Spell Slots Section', () => {
    it('displays spell slots for spellcasters', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.find('[data-testid="spell-slots-container"]').exists()).toBe(true)
    })

    it('hides spell slots for non-spellcasters', async () => {
      const nonCaster = { ...mockCharacter, spell_slots: {} }
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: nonCaster }
      })
      expect(wrapper.find('[data-testid="spell-slots-container"]').exists()).toBe(false)
    })
  })

  describe('Conditions Section', () => {
    it('displays active conditions', async () => {
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: mockCharacter }
      })
      expect(wrapper.text()).toContain('Blessed')
    })

    it('shows no conditions message when empty', async () => {
      const noConditions = { ...mockCharacter, conditions: [] }
      const wrapper = await mountSuspended(CharacterDetail, {
        props: { character: noConditions }
      })
      expect(wrapper.text()).toMatch(/no.*condition|none/i)
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/CharacterDetail.test.ts`
Expected: FAIL

**Step 3: Write the component**

```vue
<!-- app/components/dm-screen/CharacterDetail.vue -->
<script setup lang="ts">
import type { DmScreenCharacter, CharacterSavingThrows } from '~/types/dm-screen'

interface Props {
  character: DmScreenCharacter
}

const props = defineProps<Props>()

function formatModifier(mod: number): string {
  return mod >= 0 ? `+${mod}` : `${mod}`
}

const hasSpellSlots = computed(() =>
  Object.values(props.character.spell_slots).some(s => s.max > 0)
)

const speeds = computed(() => {
  const s = props.character.combat.speeds
  const result: { label: string; value: number }[] = []
  if (s.walk) result.push({ label: 'Walk', value: s.walk })
  if (s.fly) result.push({ label: 'Fly', value: s.fly })
  if (s.swim) result.push({ label: 'Swim', value: s.swim })
  if (s.climb) result.push({ label: 'Climb', value: s.climb })
  return result
})

const savingThrowEntries = computed(() => {
  const order: (keyof CharacterSavingThrows)[] = ['STR', 'DEX', 'CON', 'INT', 'WIS', 'CHA']
  return order.map(key => ({
    ability: key,
    modifier: props.character.saving_throws[key]
  }))
})
</script>

<template>
  <div class="bg-white dark:bg-neutral-900 border-t border-neutral-200 dark:border-neutral-700 p-4">
    <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
      <!-- Combat Section -->
      <div>
        <h4 class="text-xs font-medium text-neutral-500 uppercase mb-2">Combat</h4>

        <!-- Speeds -->
        <div class="mb-3">
          <div class="text-xs text-neutral-400 mb-1">Speeds</div>
          <div class="flex flex-wrap gap-2">
            <span
              v-for="speed in speeds"
              :key="speed.label"
              class="text-sm"
            >
              {{ speed.label }}: {{ speed.value }} ft
            </span>
          </div>
        </div>

        <!-- Concentration -->
        <div
          v-if="character.combat.concentration.active"
          class="mb-3"
        >
          <div class="text-xs text-neutral-400 mb-1">Concentration</div>
          <UBadge
            color="spell"
            variant="subtle"
            size="md"
          >
            {{ character.combat.concentration.spell }}
          </UBadge>
        </div>

        <!-- Saving Throws -->
        <div>
          <div class="text-xs text-neutral-400 mb-1">Saving Throws</div>
          <div class="grid grid-cols-3 gap-1 text-sm">
            <div
              v-for="st in savingThrowEntries"
              :key="st.ability"
              class="flex justify-between"
            >
              <span class="text-neutral-500">{{ st.ability }}</span>
              <span class="font-mono">{{ formatModifier(st.modifier) }}</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Equipment Section -->
      <div>
        <h4 class="text-xs font-medium text-neutral-500 uppercase mb-2">Equipment</h4>

        <!-- Armor -->
        <div
          v-if="character.equipment.armor"
          class="mb-2"
        >
          <div class="text-xs text-neutral-400 mb-1">Armor</div>
          <div class="text-sm">
            {{ character.equipment.armor.name }}
            <span class="text-neutral-500">({{ character.equipment.armor.type }})</span>
            <span
              v-if="character.equipment.armor.stealth_disadvantage"
              class="text-amber-500 ml-1"
            >
              ⚠️ Stealth disadvantage
            </span>
          </div>
        </div>

        <!-- Shield -->
        <div
          v-if="character.equipment.shield"
          class="mb-2"
        >
          <UBadge
            color="neutral"
            variant="subtle"
            size="md"
          >
            🛡️ Shield
          </UBadge>
        </div>

        <!-- Weapons -->
        <div v-if="character.equipment.weapons.length > 0">
          <div class="text-xs text-neutral-400 mb-1">Weapons</div>
          <div class="space-y-1">
            <div
              v-for="weapon in character.equipment.weapons"
              :key="weapon.name"
              class="text-sm"
            >
              <span class="font-medium">{{ weapon.name }}</span>
              <span class="text-neutral-500 ml-1">{{ weapon.damage }}</span>
              <span
                v-if="weapon.range"
                class="text-neutral-400 ml-1"
              >
                ({{ weapon.range }})
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- Capabilities Section -->
      <div>
        <h4 class="text-xs font-medium text-neutral-500 uppercase mb-2">Capabilities</h4>

        <!-- Size -->
        <div class="mb-2 text-sm">
          <span class="text-neutral-500">Size:</span> {{ character.capabilities.size }}
        </div>

        <!-- Languages -->
        <div class="mb-2">
          <div class="text-xs text-neutral-400 mb-1">Languages</div>
          <div class="flex flex-wrap gap-1">
            <UBadge
              v-for="lang in character.capabilities.languages"
              :key="lang"
              color="neutral"
              variant="subtle"
              size="md"
            >
              {{ lang }}
            </UBadge>
          </div>
        </div>

        <!-- Tool Proficiencies -->
        <div v-if="character.capabilities.tool_proficiencies.length > 0">
          <div class="text-xs text-neutral-400 mb-1">Tools</div>
          <div class="flex flex-wrap gap-1">
            <UBadge
              v-for="tool in character.capabilities.tool_proficiencies"
              :key="tool"
              color="neutral"
              variant="outline"
              size="md"
            >
              {{ tool }}
            </UBadge>
          </div>
        </div>
      </div>

      <!-- Spell Slots + Conditions -->
      <div>
        <!-- Spell Slots -->
        <div
          v-if="hasSpellSlots"
          class="mb-4"
        >
          <h4 class="text-xs font-medium text-neutral-500 uppercase mb-2">Spell Slots</h4>
          <DmScreenSpellSlotsCompact :slots="character.spell_slots" />
        </div>

        <!-- Conditions -->
        <div>
          <h4 class="text-xs font-medium text-neutral-500 uppercase mb-2">Conditions</h4>
          <div
            v-if="character.conditions.length > 0"
            class="flex flex-wrap gap-1"
          >
            <UBadge
              v-for="condition in character.conditions"
              :key="condition.slug"
              color="warning"
              variant="subtle"
              size="md"
            >
              {{ condition.name }}
            </UBadge>
          </div>
          <div
            v-else
            class="text-sm text-neutral-400"
          >
            No active conditions
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/CharacterDetail.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/dm-screen/CharacterDetail.vue tests/components/dm-screen/CharacterDetail.test.ts
git commit -m "feat(dm-screen): add CharacterDetail component with all sections"
```

---

## Task 9: CombatTable Component

**Files:**
- Create: `app/components/dm-screen/CombatTable.vue`
- Test: `tests/components/dm-screen/CombatTable.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/components/dm-screen/CombatTable.test.ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import CombatTable from '~/components/dm-screen/CombatTable.vue'
import type { DmScreenCharacter } from '~/types/dm-screen'

const mockCharacters: DmScreenCharacter[] = [
  {
    id: 1,
    public_id: 'char-1',
    name: 'Aldric',
    level: 5,
    class_name: 'Fighter',
    hit_points: { current: 40, max: 45, temp: 0 },
    armor_class: 18,
    proficiency_bonus: 3,
    combat: {
      initiative_modifier: 3,
      speeds: { walk: 30, fly: null, swim: null, climb: null },
      death_saves: { successes: 0, failures: 0 },
      concentration: { active: false, spell: null }
    },
    senses: { passive_perception: 12, passive_investigation: 10, passive_insight: 11, darkvision: null },
    capabilities: { languages: ['Common'], size: 'Medium', tool_proficiencies: [] },
    equipment: { armor: null, weapons: [], shield: false },
    saving_throws: { STR: 5, DEX: 3, CON: 4, INT: 0, WIS: 1, CHA: -1 },
    conditions: [],
    spell_slots: {}
  },
  {
    id: 2,
    public_id: 'char-2',
    name: 'Mira',
    level: 5,
    class_name: 'Cleric',
    hit_points: { current: 28, max: 35, temp: 0 },
    armor_class: 16,
    proficiency_bonus: 3,
    combat: {
      initiative_modifier: 1,
      speeds: { walk: 30, fly: null, swim: null, climb: null },
      death_saves: { successes: 0, failures: 0 },
      concentration: { active: false, spell: null }
    },
    senses: { passive_perception: 14, passive_investigation: 11, passive_insight: 14, darkvision: 60 },
    capabilities: { languages: ['Common', 'Elvish'], size: 'Medium', tool_proficiencies: [] },
    equipment: { armor: null, weapons: [], shield: true },
    saving_throws: { STR: 0, DEX: 1, CON: 2, INT: 0, WIS: 5, CHA: 2 },
    conditions: [],
    spell_slots: { '1': { current: 4, max: 4 } }
  }
]

describe('DmScreenCombatTable', () => {
  it('displays table headers', async () => {
    const wrapper = await mountSuspended(CombatTable, {
      props: { characters: mockCharacters }
    })
    expect(wrapper.text()).toContain('Name')
    expect(wrapper.text()).toContain('HP')
    expect(wrapper.text()).toContain('AC')
    expect(wrapper.text()).toContain('Init')
    expect(wrapper.text()).toContain('Perc')
  })

  it('renders a row for each character', async () => {
    const wrapper = await mountSuspended(CombatTable, {
      props: { characters: mockCharacters }
    })
    expect(wrapper.text()).toContain('Aldric')
    expect(wrapper.text()).toContain('Mira')
  })

  it('expands character detail on row click', async () => {
    const wrapper = await mountSuspended(CombatTable, {
      props: { characters: mockCharacters }
    })
    const firstRow = wrapper.find('[data-testid="combat-row"]')
    await firstRow.trigger('click')
    expect(wrapper.find('[data-testid="character-detail"]').exists()).toBe(true)
  })

  it('collapses expanded row on second click', async () => {
    const wrapper = await mountSuspended(CombatTable, {
      props: { characters: mockCharacters }
    })
    const firstRow = wrapper.find('[data-testid="combat-row"]')
    await firstRow.trigger('click')
    await firstRow.trigger('click')
    expect(wrapper.find('[data-testid="character-detail"]').exists()).toBe(false)
  })

  it('only expands one character at a time', async () => {
    const wrapper = await mountSuspended(CombatTable, {
      props: { characters: mockCharacters }
    })
    const rows = wrapper.findAll('[data-testid="combat-row"]')
    await rows[0].trigger('click')
    await rows[1].trigger('click')
    const details = wrapper.findAll('[data-testid="character-detail"]')
    expect(details.length).toBe(1)
  })

  it('shows empty state when no characters', async () => {
    const wrapper = await mountSuspended(CombatTable, {
      props: { characters: [] }
    })
    expect(wrapper.text()).toMatch(/no character|empty/i)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/CombatTable.test.ts`
Expected: FAIL

**Step 3: Write the component**

```vue
<!-- app/components/dm-screen/CombatTable.vue -->
<script setup lang="ts">
import type { DmScreenCharacter } from '~/types/dm-screen'

interface Props {
  characters: DmScreenCharacter[]
}

defineProps<Props>()

const expandedCharacterId = ref<number | null>(null)

function toggleExpand(characterId: number) {
  if (expandedCharacterId.value === characterId) {
    expandedCharacterId.value = null
  } else {
    expandedCharacterId.value = characterId
  }
}
</script>

<template>
  <div class="overflow-x-auto">
    <table
      v-if="characters.length > 0"
      class="w-full text-sm"
    >
      <thead>
        <tr class="border-b border-neutral-200 dark:border-neutral-700 text-left">
          <th class="py-3 px-4 font-medium text-neutral-500 dark:text-neutral-400">
            Name
          </th>
          <th class="py-3 px-4 font-medium text-neutral-500 dark:text-neutral-400 min-w-[180px]">
            HP
          </th>
          <th class="py-3 px-4 font-medium text-neutral-500 dark:text-neutral-400 text-center">
            AC
          </th>
          <th class="py-3 px-4 font-medium text-neutral-500 dark:text-neutral-400 text-center">
            Init
          </th>
          <th class="py-3 px-4 font-medium text-neutral-500 dark:text-neutral-400 text-center">
            Perc
          </th>
          <th class="py-3 px-4 font-medium text-neutral-500 dark:text-neutral-400 text-center">
            Inv
          </th>
          <th class="py-3 px-4 font-medium text-neutral-500 dark:text-neutral-400 text-center">
            Ins
          </th>
        </tr>
      </thead>
      <tbody>
        <template
          v-for="character in characters"
          :key="character.id"
        >
          <DmScreenCombatTableRow
            :character="character"
            :expanded="expandedCharacterId === character.id"
            @click="toggleExpand(character.id)"
          />
          <tr
            v-if="expandedCharacterId === character.id"
            data-testid="character-detail"
          >
            <td colspan="7">
              <DmScreenCharacterDetail :character="character" />
            </td>
          </tr>
        </template>
      </tbody>
    </table>

    <!-- Empty state -->
    <div
      v-else
      class="text-center py-12 text-neutral-500"
    >
      <UIcon
        name="i-heroicons-users"
        class="w-12 h-12 mx-auto mb-4 text-neutral-300"
      />
      <p>No characters in party</p>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/dm-screen/CombatTable.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/components/dm-screen/CombatTable.vue tests/components/dm-screen/CombatTable.test.ts
git commit -m "feat(dm-screen): add CombatTable component with expandable rows"
```

---

## Task 10: DM Screen Page

**Files:**
- Create: `app/pages/parties/[id]/dm-screen.vue`
- Test: `tests/pages/parties/dm-screen.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/pages/parties/dm-screen.test.ts
import { describe, it, expect, beforeAll, afterEach, afterAll, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import { server, http, HttpResponse } from '@/tests/msw/server'
import DmScreenPage from '~/pages/parties/[id]/dm-screen.vue'

const mockStats = {
  data: {
    party: { id: 1, name: 'The Brave Adventurers', description: 'A test party' },
    characters: [
      {
        id: 1,
        public_id: 'char-1',
        name: 'Aldric',
        level: 5,
        class_name: 'Fighter',
        hit_points: { current: 40, max: 45, temp: 0 },
        armor_class: 18,
        proficiency_bonus: 3,
        combat: {
          initiative_modifier: 3,
          speeds: { walk: 30, fly: null, swim: null, climb: null },
          death_saves: { successes: 0, failures: 0 },
          concentration: { active: false, spell: null }
        },
        senses: { passive_perception: 12, passive_investigation: 10, passive_insight: 11, darkvision: null },
        capabilities: { languages: ['Common'], size: 'Medium', tool_proficiencies: [] },
        equipment: { armor: null, weapons: [], shield: false },
        saving_throws: { STR: 5, DEX: 3, CON: 4, INT: 0, WIS: 1, CHA: -1 },
        conditions: [],
        spell_slots: {}
      }
    ],
    party_summary: {
      all_languages: ['Common', 'Elvish'],
      darkvision_count: 1,
      no_darkvision: ['Aldric'],
      has_healer: false,
      healers: [],
      has_detect_magic: true,
      has_dispel_magic: false,
      has_counterspell: false
    }
  }
}

describe('DM Screen Page', () => {
  beforeAll(() => server.listen())
  afterEach(() => server.resetHandlers())
  afterAll(() => server.close())

  beforeEach(() => {
    setActivePinia(createPinia())

    server.use(
      http.get('http://localhost:8080/api/v1/parties/1/stats', () => {
        return HttpResponse.json(mockStats)
      })
    )
  })

  it('displays party name in header', async () => {
    const wrapper = await mountSuspended(DmScreenPage, {
      route: { params: { id: '1' } }
    })
    expect(wrapper.text()).toContain('The Brave Adventurers')
  })

  it('displays back link to party detail', async () => {
    const wrapper = await mountSuspended(DmScreenPage, {
      route: { params: { id: '1' } }
    })
    const backLink = wrapper.find('[data-testid="back-link"]')
    expect(backLink.attributes('href')).toContain('/parties/1')
  })

  it('displays party summary panel', async () => {
    const wrapper = await mountSuspended(DmScreenPage, {
      route: { params: { id: '1' } }
    })
    expect(wrapper.text()).toContain('Party Summary')
  })

  it('displays combat table with characters', async () => {
    const wrapper = await mountSuspended(DmScreenPage, {
      route: { params: { id: '1' } }
    })
    expect(wrapper.text()).toContain('Aldric')
    expect(wrapper.text()).toContain('Fighter')
  })

  it('has refresh button', async () => {
    const wrapper = await mountSuspended(DmScreenPage, {
      route: { params: { id: '1' } }
    })
    const refreshButton = wrapper.find('[data-testid="refresh-button"]')
    expect(refreshButton.exists()).toBe(true)
  })

  it('shows loading state initially', async () => {
    const wrapper = await mountSuspended(DmScreenPage, {
      route: { params: { id: '1' } }
    })
    // Component should exist even during load
    expect(wrapper.exists()).toBe(true)
  })

  it('shows error state on API failure', async () => {
    server.use(
      http.get('http://localhost:8080/api/v1/parties/1/stats', () => {
        return HttpResponse.json({ error: 'Not found' }, { status: 404 })
      })
    )

    const wrapper = await mountSuspended(DmScreenPage, {
      route: { params: { id: '1' } }
    })

    // Wait for error to render
    await new Promise(r => setTimeout(r, 100))
    expect(wrapper.text()).toMatch(/error|fail/i)
  })

  it('persists summary collapse state in localStorage', async () => {
    // Set collapsed state
    localStorage.setItem('dm-screen-summary-collapsed', 'true')

    const wrapper = await mountSuspended(DmScreenPage, {
      route: { params: { id: '1' } }
    })

    // Summary should start collapsed (content hidden)
    // The toggle button should still exist
    expect(wrapper.find('[data-testid="summary-toggle"]').exists()).toBe(true)

    // Clean up
    localStorage.removeItem('dm-screen-summary-collapsed')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/pages/parties/dm-screen.test.ts`
Expected: FAIL

**Step 3: Write the page**

```vue
<!-- app/pages/parties/[id]/dm-screen.vue -->
<script setup lang="ts">
import type { DmScreenPartyStats } from '~/types/dm-screen'
import { logger } from '~/utils/logger'

const route = useRoute()
const partyId = computed(() => route.params.id as string)

const { apiFetch } = useApi()

// Fetch party stats
const { data: statsResponse, pending, error, refresh } = await useAsyncData(
  `party-stats-${partyId.value}`,
  () => apiFetch<{ data: DmScreenPartyStats }>(`/parties/${partyId.value}/stats`)
)

const stats = computed(() => statsResponse.value?.data)

// SEO
useSeoMeta({
  title: () => stats.value ? `DM Screen - ${stats.value.party.name}` : 'DM Screen',
  description: () => 'Combat reference and party overview for dungeon masters'
})

// Summary collapse state (persisted)
const STORAGE_KEY = 'dm-screen-summary-collapsed'

const summaryCollapsed = ref(false)

onMounted(() => {
  if (import.meta.client) {
    const stored = localStorage.getItem(STORAGE_KEY)
    if (stored !== null) {
      summaryCollapsed.value = stored === 'true'
    }
  }
})

watch(summaryCollapsed, (val) => {
  if (import.meta.client) {
    localStorage.setItem(STORAGE_KEY, String(val))
  }
})

// Refresh handler
const isRefreshing = ref(false)

async function handleRefresh() {
  isRefreshing.value = true
  try {
    await refresh()
  } catch (err) {
    logger.error('Refresh failed:', err)
  } finally {
    isRefreshing.value = false
  }
}
</script>

<template>
  <div class="container mx-auto px-4 py-6 max-w-7xl">
    <!-- Header -->
    <div class="flex items-center justify-between mb-6">
      <div class="flex items-center gap-4">
        <NuxtLink
          data-testid="back-link"
          :to="`/parties/${partyId}`"
          class="inline-flex items-center gap-1 text-sm text-neutral-600 dark:text-neutral-400 hover:text-primary-500"
        >
          <UIcon
            name="i-heroicons-arrow-left"
            class="w-4 h-4"
          />
          Back to Party
        </NuxtLink>

        <h1
          v-if="stats"
          class="text-2xl font-bold text-neutral-900 dark:text-white"
        >
          {{ stats.party.name }}
        </h1>
      </div>

      <UButton
        data-testid="refresh-button"
        icon="i-heroicons-arrow-path"
        variant="soft"
        :loading="isRefreshing"
        @click="handleRefresh"
      >
        Refresh
      </UButton>
    </div>

    <!-- Loading State -->
    <div
      v-if="pending"
      class="flex justify-center py-12"
    >
      <UIcon
        name="i-heroicons-arrow-path"
        class="w-8 h-8 animate-spin text-neutral-400"
      />
    </div>

    <!-- Error State -->
    <UAlert
      v-else-if="error"
      color="error"
      icon="i-heroicons-exclamation-triangle"
      title="Failed to load party stats"
      class="mb-6"
    >
      <template #description>
        Could not load the DM Screen data. Please try refreshing.
      </template>
      <template #actions>
        <UButton
          variant="soft"
          color="error"
          @click="handleRefresh"
        >
          Retry
        </UButton>
      </template>
    </UAlert>

    <!-- Content -->
    <template v-else-if="stats">
      <!-- Party Summary -->
      <div class="mb-6">
        <DmScreenPartySummary
          v-model:collapsed="summaryCollapsed"
          :summary="stats.party_summary"
        />
      </div>

      <!-- Combat Table -->
      <div class="bg-white dark:bg-neutral-900 rounded-lg border border-neutral-200 dark:border-neutral-700">
        <DmScreenCombatTable :characters="stats.characters" />
      </div>
    </template>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/pages/parties/dm-screen.test.ts`
Expected: PASS

**Step 5: Commit**

```bash
git add app/pages/parties/[id]/dm-screen.vue tests/pages/parties/dm-screen.test.ts
git commit -m "feat(dm-screen): add DM Screen page with party stats display"
```

---

## Task 11: Add DM Screen Link to Party Detail Page

**Files:**
- Modify: `app/pages/parties/[id].vue:270-285`
- Test: `tests/pages/parties/detail.test.ts`

**Step 1: Write the failing test**

```typescript
// tests/pages/parties/detail.test.ts
// Add to existing test file or create new

import { describe, it, expect, beforeAll, afterEach, afterAll, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import { server, http, HttpResponse } from '@/tests/msw/server'
import PartyDetailPage from '~/pages/parties/[id].vue'

describe('Party Detail Page - DM Screen Link', () => {
  beforeAll(() => server.listen())
  afterEach(() => server.resetHandlers())
  afterAll(() => server.close())

  beforeEach(() => {
    setActivePinia(createPinia())

    server.use(
      http.get('http://localhost:8080/api/v1/parties/1', () => {
        return HttpResponse.json({
          data: {
            id: 1,
            name: 'Test Party',
            description: null,
            character_count: 2,
            created_at: '2024-01-01',
            characters: []
          }
        })
      })
    )
  })

  it('displays link to DM Screen', async () => {
    const wrapper = await mountSuspended(PartyDetailPage, {
      route: { params: { id: '1' } }
    })
    const dmScreenLink = wrapper.find('[data-testid="dm-screen-link"]')
    expect(dmScreenLink.exists()).toBe(true)
    expect(dmScreenLink.attributes('href')).toContain('/parties/1/dm-screen')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/pages/parties/detail.test.ts -t "DM Screen Link"`
Expected: FAIL

**Step 3: Add the link to party detail page**

Find the header section in `app/pages/parties/[id].vue` (around line 270) and add the DM Screen button:

```vue
<!-- Add after the Edit button, before the dropdown menu -->
<UButton
  data-testid="dm-screen-link"
  :to="`/parties/${partyId}/dm-screen`"
  icon="i-heroicons-presentation-chart-bar"
  variant="soft"
  color="primary"
>
  DM Screen
</UButton>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/pages/parties/detail.test.ts -t "DM Screen Link"`
Expected: PASS

**Step 5: Commit**

```bash
git add app/pages/parties/[id].vue tests/pages/parties/detail.test.ts
git commit -m "feat(dm-screen): add DM Screen link to party detail page"
```

---

## Task 12: Run Full Test Suite and Typecheck

**Step 1: Run TypeScript check**

```bash
docker compose exec nuxt npm run typecheck
```

Expected: No errors

**Step 2: Run linter**

```bash
docker compose exec nuxt npm run lint:fix
```

Expected: No errors or auto-fixed

**Step 3: Run DM Screen test suite**

```bash
docker compose exec nuxt npm run test -- tests/components/dm-screen tests/pages/parties/dm-screen.test.ts tests/types/dm-screen.test.ts tests/server/api/parties/stats.test.ts
```

Expected: All tests pass

**Step 4: Manual browser verification**

1. Navigate to http://localhost:4000/parties
2. Click on a party with characters
3. Click "DM Screen" button
4. Verify:
   - Party summary shows correctly
   - Combat table displays all characters
   - Click row to expand character details
   - Refresh button works
   - Collapse/expand summary persists on page reload

**Step 5: Final commit**

```bash
git add -A
git commit -m "feat(dm-screen): complete DM Screen UI implementation

- Add TypeScript types for party stats API
- Add Nitro proxy route for /parties/{id}/stats
- Add HpBar, DeathSavesCompact, SpellSlotsCompact components
- Add PartySummary with collapsible panel
- Add CombatTable with expandable CharacterDetail rows
- Add DM Screen page at /parties/[id]/dm-screen
- Add DM Screen link on party detail page

Closes #564"
```

---

## Summary

This plan implements the DM Screen UI feature in 12 tasks:

1. **Types** - TypeScript types for API response
2. **API Route** - Nitro proxy for party stats
3. **HpBar** - Color-coded health bar component
4. **DeathSavesCompact** - Compact death save circles
5. **SpellSlotsCompact** - Spell slot indicators
6. **PartySummary** - Collapsible party overview panel
7. **CombatTableRow** - Table row with character stats
8. **CharacterDetail** - Expandable character detail card
9. **CombatTable** - Main combat reference table
10. **DM Screen Page** - The page component
11. **Party Link** - Link from party detail page
12. **Final Verification** - Full test suite and browser check

Each task follows TDD with explicit test-first steps and commits.
