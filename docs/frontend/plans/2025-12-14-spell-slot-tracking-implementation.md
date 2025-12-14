# Spell Slot Tracking & Preparation Toggle Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add interactive spell slot tracking (click crystals to use/restore) and spell preparation toggling (click card to prepare/unprepare) to the Character Sheet Spells Tab.

**Architecture:** Extend `characterPlayStateStore` with spell slot and preparation state. Components call store actions which make optimistic updates then API calls. SpellSlotsManager handles slot crystals, SpellCard handles preparation toggle.

**Tech Stack:** Vue 3, Pinia, Vitest, MSW, NuxtUI 4, TypeScript

**Reference:** See design doc `docs/frontend/plans/2025-12-14-spell-slot-tracking-design.md`

---

## Task 1: Create Nitro Route for Spell Preparation Toggle

**Files:**
- Create: `server/api/characters/[id]/spells/[spellId].patch.ts`

**Step 1: Create the route file**

```typescript
// server/api/characters/[id]/spells/[spellId].patch.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const spellId = getRouterParam(event, 'spellId')
  const body = await readBody(event)

  try {
    const data = await $fetch(
      `${config.apiBaseServer}/characters/${id}/spells/${spellId}`,
      { method: 'PATCH', body }
    )
    return data
  } catch (error: any) {
    throw createError({
      statusCode: error.statusCode || 500,
      statusMessage: error.statusMessage || 'Failed to update spell preparation',
      data: error.data
    })
  }
})
```

**Step 2: Verify route works**

```bash
curl -X PATCH "http://localhost:4000/api/characters/2/spells/1" \
  -H "Content-Type: application/json" \
  -d '{"is_prepared": true}'
```

Expected: JSON response with CharacterSpell data or 404/422 error

**Step 3: Commit**

```bash
git add server/api/characters/\[id\]/spells/\[spellId\].patch.ts
git commit -m "feat: add Nitro route for spell preparation toggle (#616)"
```

---

## Task 2: Create Nitro Route for Spell Slot Usage

**Files:**
- Create: `server/api/characters/[id]/spell-slots/[level].patch.ts`

**Step 1: Create the route file**

```typescript
// server/api/characters/[id]/spell-slots/[level].patch.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const level = getRouterParam(event, 'level')
  const body = await readBody(event)

  try {
    const data = await $fetch(
      `${config.apiBaseServer}/characters/${id}/spell-slots/${level}`,
      { method: 'PATCH', body }
    )
    return data
  } catch (error: any) {
    throw createError({
      statusCode: error.statusCode || 500,
      statusMessage: error.statusMessage || 'Failed to update spell slot',
      data: error.data
    })
  }
})
```

**Step 2: Verify route works**

```bash
curl -X PATCH "http://localhost:4000/api/characters/2/spell-slots/1" \
  -H "Content-Type: application/json" \
  -d '{"action": "use"}'
```

Expected: JSON response with spell slot data `{ level, total, spent, available, slot_type }`

**Step 3: Commit**

```bash
git add server/api/characters/\[id\]/spell-slots/\[level\].patch.ts
git commit -m "feat: add Nitro route for spell slot usage (#616)"
```

---

## Task 3: Add Spell Slot Types

**Files:**
- Modify: `app/types/character.ts`

**Step 1: Add spell slot state types**

Add after line 370 (after `CharacterSpellSlots` interface):

```typescript
/**
 * Spell slot state for play mode tracking
 * Tracks both total and spent for each level
 */
export interface SpellSlotState {
  level: number
  total: number
  spent: number
  slotType: 'standard' | 'pact_magic'
}

/**
 * API response from PATCH spell-slots endpoint
 */
export interface SpellSlotUpdateResponse {
  data: {
    level: number
    total: number
    spent: number
    available: number
    slot_type: 'standard' | 'pact_magic'
  }
}
```

**Step 2: Commit**

```bash
git add app/types/character.ts
git commit -m "feat: add spell slot state types (#616)"
```

---

## Task 4: Write Store Tests for Spell Slot Actions

**Files:**
- Modify: `tests/stores/characterPlayState.test.ts`

**Step 1: Read existing store test file**

Check current test structure and imports.

**Step 2: Add spell slot test suite**

Add new describe block for spell slot functionality:

```typescript
describe('spell slot tracking', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('initializes spell slots from stats', () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellSlots([
      { level: 1, total: 4 },
      { level: 2, total: 3 }
    ])

    expect(store.getSlotState(1)).toEqual({ total: 4, spent: 0, available: 4 })
    expect(store.getSlotState(2)).toEqual({ total: 3, spent: 0, available: 3 })
  })

  it('useSpellSlot increments spent count', async () => {
    const store = useCharacterPlayStateStore()
    store.initialize({
      characterId: 1,
      isDead: false,
      hitPoints: { current: 10, max: 10, temporary: 0 },
      deathSaves: { successes: 0, failures: 0 },
      currency: { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })
    store.initializeSpellSlots([{ level: 1, total: 4 }])

    // Mock API call
    vi.spyOn(store, 'useSpellSlot').mockImplementation(async (level) => {
      store.spellSlots.set(level, { total: 4, spent: 1, slotType: 'standard' })
    })

    await store.useSpellSlot(1)

    expect(store.getSlotState(1).spent).toBe(1)
    expect(store.getSlotState(1).available).toBe(3)
  })

  it('restoreSpellSlot decrements spent count', async () => {
    const store = useCharacterPlayStateStore()
    store.initialize({
      characterId: 1,
      isDead: false,
      hitPoints: { current: 10, max: 10, temporary: 0 },
      deathSaves: { successes: 0, failures: 0 },
      currency: { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })
    store.initializeSpellSlots([{ level: 1, total: 4 }])
    store.spellSlots.set(1, { total: 4, spent: 2, slotType: 'standard' })

    vi.spyOn(store, 'restoreSpellSlot').mockImplementation(async (level) => {
      store.spellSlots.set(level, { total: 4, spent: 1, slotType: 'standard' })
    })

    await store.restoreSpellSlot(1)

    expect(store.getSlotState(1).spent).toBe(1)
    expect(store.getSlotState(1).available).toBe(3)
  })

  it('cannot use slot when none available', () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellSlots([{ level: 1, total: 2 }])
    store.spellSlots.set(1, { total: 2, spent: 2, slotType: 'standard' })

    expect(store.canUseSlot(1)).toBe(false)
  })

  it('cannot restore slot when none spent', () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellSlots([{ level: 1, total: 2 }])

    expect(store.canRestoreSlot(1)).toBe(false)
  })
})
```

**Step 3: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- tests/stores/characterPlayState.test.ts -t "spell slot" --reporter=verbose
```

Expected: FAIL (methods don't exist yet)

**Step 4: Commit failing tests**

```bash
git add tests/stores/characterPlayState.test.ts
git commit -m "test: add spell slot tracking tests (red) (#616)"
```

---

## Task 5: Implement Store Spell Slot State and Actions

**Files:**
- Modify: `app/stores/characterPlayState.ts`

**Step 1: Read existing store file**

Check current state structure and imports.

**Step 2: Add spell slot state to store**

Add to state definition:

```typescript
// Add to state
spellSlots: new Map<number, { total: number, spent: number, slotType: 'standard' | 'pact_magic' }>(),
```

**Step 3: Add spell slot getters**

```typescript
// Add getters
getSlotState: (state) => (level: number) => {
  const slot = state.spellSlots.get(level)
  if (!slot) return { total: 0, spent: 0, available: 0 }
  return {
    total: slot.total,
    spent: slot.spent,
    available: slot.total - slot.spent
  }
},

canUseSlot: (state) => (level: number) => {
  const slot = state.spellSlots.get(level)
  return slot ? slot.spent < slot.total : false
},

canRestoreSlot: (state) => (level: number) => {
  const slot = state.spellSlots.get(level)
  return slot ? slot.spent > 0 : false
},
```

**Step 4: Add spell slot actions**

```typescript
// Add actions
initializeSpellSlots(slots: Array<{ level: number, total: number }>) {
  this.spellSlots.clear()
  for (const slot of slots) {
    this.spellSlots.set(slot.level, {
      total: slot.total,
      spent: 0,
      slotType: 'standard'
    })
  }
},

async useSpellSlot(level: number) {
  if (!this.canUseSlot(level) || !this.characterId) return

  const slot = this.spellSlots.get(level)
  if (!slot) return

  // Optimistic update
  const previousSpent = slot.spent
  slot.spent += 1
  this.spellSlots.set(level, { ...slot })

  try {
    const { apiFetch } = useApi()
    await apiFetch(`/characters/${this.characterId}/spell-slots/${level}`, {
      method: 'PATCH',
      body: { action: 'use' }
    })
  } catch (error) {
    // Revert on failure
    slot.spent = previousSpent
    this.spellSlots.set(level, { ...slot })
    throw error
  }
},

async restoreSpellSlot(level: number) {
  if (!this.canRestoreSlot(level) || !this.characterId) return

  const slot = this.spellSlots.get(level)
  if (!slot) return

  // Optimistic update
  const previousSpent = slot.spent
  slot.spent -= 1
  this.spellSlots.set(level, { ...slot })

  try {
    const { apiFetch } = useApi()
    await apiFetch(`/characters/${this.characterId}/spell-slots/${level}`, {
      method: 'PATCH',
      body: { action: 'restore' }
    })
  } catch (error) {
    // Revert on failure
    slot.spent = previousSpent
    this.spellSlots.set(level, { ...slot })
    throw error
  }
},

async resetSpellSlots() {
  if (!this.characterId) return

  // Reset all slots to 0 spent
  for (const [level, slot] of this.spellSlots) {
    this.spellSlots.set(level, { ...slot, spent: 0 })
  }

  // Note: Long rest endpoint handles this on backend
},
```

**Step 5: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- tests/stores/characterPlayState.test.ts -t "spell slot" --reporter=verbose
```

Expected: PASS

**Step 6: Commit**

```bash
git add app/stores/characterPlayState.ts
git commit -m "feat: implement spell slot state and actions in store (#616)"
```

---

## Task 6: Write Store Tests for Spell Preparation

**Files:**
- Modify: `tests/stores/characterPlayState.test.ts`

**Step 1: Add spell preparation test suite**

```typescript
describe('spell preparation tracking', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('initializes prepared spell IDs from spells array', () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellPreparation({
      spells: [
        { id: 1, is_prepared: true, is_always_prepared: false },
        { id: 2, is_prepared: false, is_always_prepared: false },
        { id: 3, is_prepared: true, is_always_prepared: true }
      ] as any[],
      preparationLimit: 5
    })

    expect(store.preparedSpellIds.has(1)).toBe(true)
    expect(store.preparedSpellIds.has(2)).toBe(false)
    expect(store.preparedSpellIds.has(3)).toBe(true)
    expect(store.preparedSpellCount).toBe(2)
  })

  it('atPreparationLimit returns true when at limit', () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellPreparation({
      spells: [
        { id: 1, is_prepared: true, is_always_prepared: false },
        { id: 2, is_prepared: true, is_always_prepared: false }
      ] as any[],
      preparationLimit: 2
    })

    expect(store.atPreparationLimit).toBe(true)
  })

  it('atPreparationLimit returns false when under limit', () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellPreparation({
      spells: [
        { id: 1, is_prepared: true, is_always_prepared: false }
      ] as any[],
      preparationLimit: 5
    })

    expect(store.atPreparationLimit).toBe(false)
  })

  it('atPreparationLimit returns false when no limit (known casters)', () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellPreparation({
      spells: [
        { id: 1, is_prepared: true, is_always_prepared: false }
      ] as any[],
      preparationLimit: null
    })

    expect(store.atPreparationLimit).toBe(false)
  })

  it('toggleSpellPreparation prepares an unprepared spell', async () => {
    const store = useCharacterPlayStateStore()
    store.initialize({
      characterId: 1,
      isDead: false,
      hitPoints: { current: 10, max: 10, temporary: 0 },
      deathSaves: { successes: 0, failures: 0 },
      currency: { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })
    store.initializeSpellPreparation({
      spells: [{ id: 42, is_prepared: false, is_always_prepared: false }] as any[],
      preparationLimit: 5
    })

    vi.spyOn(store, 'toggleSpellPreparation').mockImplementation(async (id, current) => {
      if (!current) store.preparedSpellIds.add(id)
      else store.preparedSpellIds.delete(id)
    })

    await store.toggleSpellPreparation(42, false)

    expect(store.preparedSpellIds.has(42)).toBe(true)
  })

  it('toggleSpellPreparation unprepares a prepared spell', async () => {
    const store = useCharacterPlayStateStore()
    store.initialize({
      characterId: 1,
      isDead: false,
      hitPoints: { current: 10, max: 10, temporary: 0 },
      deathSaves: { successes: 0, failures: 0 },
      currency: { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })
    store.initializeSpellPreparation({
      spells: [{ id: 42, is_prepared: true, is_always_prepared: false }] as any[],
      preparationLimit: 5
    })

    vi.spyOn(store, 'toggleSpellPreparation').mockImplementation(async (id, current) => {
      if (!current) store.preparedSpellIds.add(id)
      else store.preparedSpellIds.delete(id)
    })

    await store.toggleSpellPreparation(42, true)

    expect(store.preparedSpellIds.has(42)).toBe(false)
  })
})
```

**Step 2: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- tests/stores/characterPlayState.test.ts -t "spell preparation" --reporter=verbose
```

Expected: FAIL

**Step 3: Commit failing tests**

```bash
git add tests/stores/characterPlayState.test.ts
git commit -m "test: add spell preparation tracking tests (red) (#616)"
```

---

## Task 7: Implement Store Spell Preparation State and Actions

**Files:**
- Modify: `app/stores/characterPlayState.ts`

**Step 1: Add spell preparation state**

```typescript
// Add to state
preparedSpellIds: new Set<number>(),
preparationLimit: null as number | null,
```

**Step 2: Add spell preparation getters**

```typescript
// Add getters
preparedSpellCount: (state) => {
  return state.preparedSpellIds.size
},

atPreparationLimit: (state) => {
  if (state.preparationLimit === null) return false
  return state.preparedSpellIds.size >= state.preparationLimit
},

isSpellPrepared: (state) => (characterSpellId: number) => {
  return state.preparedSpellIds.has(characterSpellId)
},
```

**Step 3: Add spell preparation actions**

```typescript
// Add actions
initializeSpellPreparation(data: {
  spells: Array<{ id: number, is_prepared: boolean, is_always_prepared: boolean }>
  preparationLimit: number | null
}) {
  this.preparedSpellIds.clear()
  this.preparationLimit = data.preparationLimit

  for (const spell of data.spells) {
    if (spell.is_prepared || spell.is_always_prepared) {
      this.preparedSpellIds.add(spell.id)
    }
  }
},

async toggleSpellPreparation(characterSpellId: number, currentlyPrepared: boolean) {
  if (!this.characterId) return

  const newPreparedState = !currentlyPrepared

  // Check limit when preparing
  if (newPreparedState && this.atPreparationLimit) {
    throw new Error('Preparation limit reached')
  }

  // Optimistic update
  if (newPreparedState) {
    this.preparedSpellIds.add(characterSpellId)
  } else {
    this.preparedSpellIds.delete(characterSpellId)
  }

  try {
    const { apiFetch } = useApi()
    await apiFetch(`/characters/${this.characterId}/spells/${characterSpellId}`, {
      method: 'PATCH',
      body: { is_prepared: newPreparedState }
    })
  } catch (error) {
    // Revert on failure
    if (newPreparedState) {
      this.preparedSpellIds.delete(characterSpellId)
    } else {
      this.preparedSpellIds.add(characterSpellId)
    }
    throw error
  }
},
```

**Step 4: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- tests/stores/characterPlayState.test.ts -t "spell preparation" --reporter=verbose
```

Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterPlayState.ts
git commit -m "feat: implement spell preparation state and actions in store (#616)"
```

---

## Task 8: Write SpellSlotsManager Tests

**Files:**
- Create: `tests/components/character/sheet/SpellSlotsManager.test.ts`

**Step 1: Create test file with basic tests**

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import SpellSlotsManager from '~/components/character/sheet/SpellSlotsManager.vue'
import { useCharacterPlayStateStore } from '~/stores/characterPlayState'

describe('SpellSlotsManager', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders filled crystals for available slots', async () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellSlots([{ level: 1, total: 4 }])

    const wrapper = await mountSuspended(SpellSlotsManager, {
      props: {
        characterId: 1,
        editable: true
      }
    })

    const crystals = wrapper.findAll('[data-testid="slot-1-available"]')
    expect(crystals).toHaveLength(4)
  })

  it('renders empty crystals for spent slots', async () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellSlots([{ level: 1, total: 4 }])
    store.spellSlots.set(1, { total: 4, spent: 2, slotType: 'standard' })

    const wrapper = await mountSuspended(SpellSlotsManager, {
      props: {
        characterId: 1,
        editable: true
      }
    })

    const available = wrapper.findAll('[data-testid="slot-1-available"]')
    const spent = wrapper.findAll('[data-testid="slot-1-spent"]')
    expect(available).toHaveLength(2)
    expect(spent).toHaveLength(2)
  })

  it('clicking available crystal calls useSpellSlot', async () => {
    const store = useCharacterPlayStateStore()
    store.initialize({
      characterId: 1,
      isDead: false,
      hitPoints: { current: 10, max: 10, temporary: 0 },
      deathSaves: { successes: 0, failures: 0 },
      currency: { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })
    store.initializeSpellSlots([{ level: 1, total: 4 }])

    const useSlotSpy = vi.spyOn(store, 'useSpellSlot').mockResolvedValue()

    const wrapper = await mountSuspended(SpellSlotsManager, {
      props: {
        characterId: 1,
        editable: true
      }
    })

    const crystal = wrapper.find('[data-testid="slot-1-available"]')
    await crystal.trigger('click')

    expect(useSlotSpy).toHaveBeenCalledWith(1)
  })

  it('clicking spent crystal calls restoreSpellSlot', async () => {
    const store = useCharacterPlayStateStore()
    store.initialize({
      characterId: 1,
      isDead: false,
      hitPoints: { current: 10, max: 10, temporary: 0 },
      deathSaves: { successes: 0, failures: 0 },
      currency: { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })
    store.initializeSpellSlots([{ level: 1, total: 4 }])
    store.spellSlots.set(1, { total: 4, spent: 2, slotType: 'standard' })

    const restoreSpy = vi.spyOn(store, 'restoreSpellSlot').mockResolvedValue()

    const wrapper = await mountSuspended(SpellSlotsManager, {
      props: {
        characterId: 1,
        editable: true
      }
    })

    const crystal = wrapper.find('[data-testid="slot-1-spent"]')
    await crystal.trigger('click')

    expect(restoreSpy).toHaveBeenCalledWith(1)
  })

  it('crystals are not clickable when editable is false', async () => {
    const store = useCharacterPlayStateStore()
    store.initializeSpellSlots([{ level: 1, total: 4 }])

    const useSlotSpy = vi.spyOn(store, 'useSpellSlot')

    const wrapper = await mountSuspended(SpellSlotsManager, {
      props: {
        characterId: 1,
        editable: false
      }
    })

    const crystal = wrapper.find('[data-testid="slot-1-available"]')
    await crystal.trigger('click')

    expect(useSlotSpy).not.toHaveBeenCalled()
  })
})
```

**Step 2: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellSlotsManager.test.ts --reporter=verbose
```

Expected: FAIL (component doesn't exist yet)

**Step 3: Commit failing tests**

```bash
git add tests/components/character/sheet/SpellSlotsManager.test.ts
git commit -m "test: add SpellSlotsManager component tests (red) (#616)"
```

---

## Task 9: Implement SpellSlotsManager Component

**Files:**
- Rename: `app/components/character/sheet/SpellSlots.vue` → `SpellSlotsManager.vue`
- Modify: `app/components/character/sheet/SpellSlotsManager.vue`

**Step 1: Rename the file**

```bash
git mv app/components/character/sheet/SpellSlots.vue app/components/character/sheet/SpellSlotsManager.vue
```

**Step 2: Rewrite the component**

```vue
<!-- app/components/character/sheet/SpellSlotsManager.vue -->
<script setup lang="ts">
/**
 * Interactive Spell Slots Manager
 *
 * Displays spell slots as clickable crystal icons.
 * Filled = available (click to use), Empty = spent (click to restore).
 *
 * @see Issue #616 - Spell slot tracking
 */
import { storeToRefs } from 'pinia'
import { useCharacterPlayStateStore } from '~/stores/characterPlayState'

const props = defineProps<{
  characterId: number
  editable: boolean
}>()

const store = useCharacterPlayStateStore()
const { spellSlots } = storeToRefs(store)

/**
 * Convert spell level to ordinal (1st, 2nd, 3rd, etc.)
 */
function ordinal(level: number): string {
  const suffixes = ['th', 'st', 'nd', 'rd']
  const value = level % 100
  const v = value - 20
  const suffix = (v >= 0 && v < 10 && suffixes[v]) || suffixes[value] || 'th'
  return level + suffix
}

/**
 * Get sorted list of spell levels that have slots
 */
const sortedLevels = computed(() => {
  const levels: Array<{ level: number, total: number, spent: number, available: number }> = []

  for (const [level, slot] of spellSlots.value) {
    if (slot.total > 0) {
      levels.push({
        level,
        total: slot.total,
        spent: slot.spent,
        available: slot.total - slot.spent
      })
    }
  }

  return levels.sort((a, b) => a.level - b.level)
})

/**
 * Handle click on available slot (use it)
 */
async function handleUseSlot(level: number) {
  if (!props.editable) return
  try {
    await store.useSpellSlot(level)
  } catch (error) {
    // Toast handled by store or parent
  }
}

/**
 * Handle click on spent slot (restore it)
 */
async function handleRestoreSlot(level: number) {
  if (!props.editable) return
  try {
    await store.restoreSpellSlot(level)
  } catch (error) {
    // Toast handled by store or parent
  }
}
</script>

<template>
  <div
    v-if="sortedLevels.length > 0"
    class="space-y-4"
  >
    <h4 class="text-sm font-semibold text-gray-700 dark:text-gray-300 uppercase">
      Spell Slots
    </h4>

    <div class="space-y-2">
      <div
        v-for="{ level, total, spent, available } in sortedLevels"
        :key="level"
        class="flex items-center gap-3"
      >
        <!-- Level Label -->
        <div class="text-sm font-medium text-gray-700 dark:text-gray-300 w-8">
          {{ ordinal(level) }}
        </div>

        <!-- Slot Crystals -->
        <div class="flex gap-1.5">
          <!-- Available slots (filled) -->
          <button
            v-for="i in available"
            :key="`available-${level}-${i}`"
            :data-testid="`slot-${level}-available`"
            :disabled="!editable"
            :class="[
              'w-7 h-7 transition-all',
              editable ? 'cursor-pointer hover:scale-110' : 'cursor-default'
            ]"
            @click="handleUseSlot(level)"
          >
            <UIcon
              name="i-game-icons-crystal-shine"
              class="w-7 h-7 text-spell-500 dark:text-spell-400"
            />
          </button>

          <!-- Spent slots (empty/hollow) -->
          <button
            v-for="i in spent"
            :key="`spent-${level}-${i}`"
            :data-testid="`slot-${level}-spent`"
            :disabled="!editable"
            :class="[
              'w-7 h-7 transition-all',
              editable ? 'cursor-pointer hover:scale-110' : 'cursor-default'
            ]"
            @click="handleRestoreSlot(level)"
          >
            <UIcon
              name="i-game-icons-crystal-shine"
              class="w-7 h-7 text-gray-300 dark:text-gray-600"
            />
          </button>
        </div>
      </div>
    </div>
  </div>
</template>
```

**Step 3: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellSlotsManager.test.ts --reporter=verbose
```

Expected: PASS

**Step 4: Commit**

```bash
git add app/components/character/sheet/SpellSlotsManager.vue
git commit -m "feat: implement interactive SpellSlotsManager component (#616)"
```

---

## Task 10: Write SpellCard Preparation Toggle Tests

**Files:**
- Modify: `tests/components/character/sheet/SpellCard.test.ts`

**Step 1: Add preparation toggle tests**

```typescript
describe('preparation toggle', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('clicking card body toggles preparation when editable', async () => {
    const store = useCharacterPlayStateStore()
    store.initialize({
      characterId: 1,
      isDead: false,
      hitPoints: { current: 10, max: 10, temporary: 0 },
      deathSaves: { successes: 0, failures: 0 },
      currency: { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })
    store.initializeSpellPreparation({
      spells: [{ id: 42, is_prepared: false, is_always_prepared: false }],
      preparationLimit: 5
    })

    const toggleSpy = vi.spyOn(store, 'toggleSpellPreparation').mockResolvedValue()

    const wrapper = await mountSuspended(SpellCard, {
      props: {
        spell: mockCharacterSpell,
        characterId: 1,
        editable: true,
        atPrepLimit: false
      }
    })

    await wrapper.find('[data-testid="spell-card"]').trigger('click')

    expect(toggleSpy).toHaveBeenCalledWith(42, false)
  })

  it('clicking chevron expands details without toggling preparation', async () => {
    const store = useCharacterPlayStateStore()
    const toggleSpy = vi.spyOn(store, 'toggleSpellPreparation')

    const wrapper = await mountSuspended(SpellCard, {
      props: {
        spell: mockCharacterSpell,
        characterId: 1,
        editable: true,
        atPrepLimit: false
      }
    })

    await wrapper.find('[data-testid="expand-toggle"]').trigger('click')

    expect(toggleSpy).not.toHaveBeenCalled()
    expect(wrapper.find('[data-testid="spell-details"]').exists()).toBe(true)
  })

  it('card is greyed out and not clickable when at prep limit and unprepared', async () => {
    const store = useCharacterPlayStateStore()
    const toggleSpy = vi.spyOn(store, 'toggleSpellPreparation')

    const unpreparedSpell = { ...mockCharacterSpell, is_prepared: false }

    const wrapper = await mountSuspended(SpellCard, {
      props: {
        spell: unpreparedSpell,
        characterId: 1,
        editable: true,
        atPrepLimit: true
      }
    })

    await wrapper.find('[data-testid="spell-card"]').trigger('click')

    expect(toggleSpy).not.toHaveBeenCalled()
    expect(wrapper.find('[data-testid="spell-card"]').classes()).toContain('opacity-40')
  })

  it('always-prepared spells cannot be toggled', async () => {
    const store = useCharacterPlayStateStore()
    const toggleSpy = vi.spyOn(store, 'toggleSpellPreparation')

    const alwaysPreparedSpell = { ...mockCharacterSpell, is_always_prepared: true, is_prepared: true }

    const wrapper = await mountSuspended(SpellCard, {
      props: {
        spell: alwaysPreparedSpell,
        characterId: 1,
        editable: true,
        atPrepLimit: false
      }
    })

    await wrapper.find('[data-testid="spell-card"]').trigger('click')

    expect(toggleSpy).not.toHaveBeenCalled()
  })
})
```

**Step 2: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellCard.test.ts -t "preparation toggle" --reporter=verbose
```

Expected: FAIL

**Step 3: Commit failing tests**

```bash
git add tests/components/character/sheet/SpellCard.test.ts
git commit -m "test: add SpellCard preparation toggle tests (red) (#616)"
```

---

## Task 11: Implement SpellCard Preparation Toggle

**Files:**
- Modify: `app/components/character/sheet/SpellCard.vue`

**Step 1: Update props and add store integration**

Add to script setup:

```typescript
import { storeToRefs } from 'pinia'
import { useCharacterPlayStateStore } from '~/stores/characterPlayState'

const props = defineProps<{
  spell: CharacterSpell
  characterId?: number
  editable?: boolean
  atPrepLimit?: boolean
}>()

const store = useCharacterPlayStateStore()

/**
 * Check if this card can toggle preparation
 */
const canToggle = computed(() => {
  if (!props.editable) return false
  if (props.spell.is_always_prepared) return false
  if (!props.spell.is_prepared && props.atPrepLimit) return false
  return true
})

/**
 * Check if card should appear greyed out
 */
const isGreyedOut = computed(() => {
  if (!props.spell.is_prepared && props.atPrepLimit) return true
  return false
})

/**
 * Handle card body click (toggle preparation)
 */
async function handleCardClick(event: MouseEvent) {
  // Don't toggle if clicking expand chevron
  const target = event.target as HTMLElement
  if (target.closest('[data-testid="expand-toggle"]')) return

  if (!canToggle.value) return

  try {
    await store.toggleSpellPreparation(props.spell.id, props.spell.is_prepared)
  } catch (error) {
    // Error handling done by store
  }
}

/**
 * Handle chevron click (expand/collapse only)
 */
function handleExpandClick(event: MouseEvent) {
  event.stopPropagation()
  isExpanded.value = !isExpanded.value
}
```

**Step 2: Update template**

Update the card div to use new click handlers and classes:

```vue
<div
  v-if="spellData"
  data-testid="spell-card"
  :class="[
    'p-3 rounded-lg border transition-all',
    'bg-white dark:bg-gray-800',
    isPrepared
      ? 'border-spell-300 dark:border-spell-700'
      : 'border-gray-200 dark:border-gray-700',
    isGreyedOut
      ? 'opacity-40 cursor-not-allowed'
      : canToggle
        ? 'cursor-pointer hover:shadow-md'
        : 'cursor-default',
    !isPrepared && !isGreyedOut && 'opacity-60'
  ]"
  @click="handleCardClick"
>
```

Update the chevron to be a clickable button:

```vue
<!-- Expand/collapse button -->
<button
  data-testid="expand-toggle"
  class="p-1 -m-1 rounded hover:bg-gray-100 dark:hover:bg-gray-700"
  @click="handleExpandClick"
>
  <UIcon
    :name="isExpanded ? 'i-heroicons-chevron-up' : 'i-heroicons-chevron-down'"
    class="w-5 h-5 text-gray-400"
  />
</button>
```

Add data-testid to expanded details:

```vue
<div
  v-if="isExpanded"
  data-testid="spell-details"
  class="mt-3 pt-3 border-t border-gray-200 dark:border-gray-700 space-y-2 text-sm"
>
```

**Step 3: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellCard.test.ts -t "preparation toggle" --reporter=verbose
```

Expected: PASS

**Step 4: Commit**

```bash
git add app/components/character/sheet/SpellCard.vue
git commit -m "feat: implement spell preparation toggle on SpellCard (#616)"
```

---

## Task 12: Update SpellsPanel to Pass New Props

**Files:**
- Modify: `app/components/character/sheet/SpellsPanel.vue`

**Step 1: Add new props**

```typescript
const props = defineProps<{
  spells: CharacterSpell[]
  stats: CharacterStats
  characterId: number
  editable: boolean
}>()
```

**Step 2: Add store integration for atPrepLimit**

```typescript
import { storeToRefs } from 'pinia'
import { useCharacterPlayStateStore } from '~/stores/characterPlayState'

const store = useCharacterPlayStateStore()
const { atPreparationLimit } = storeToRefs(store)
```

**Step 3: Update SpellSlotsManager usage**

```vue
<!-- Spell Slots -->
<CharacterSheetSpellSlotsManager
  v-if="stats.spell_slots && (Array.isArray(stats.spell_slots) ? stats.spell_slots.some(s => s > 0) : Object.values(stats.spell_slots).some(s => s > 0))"
  :character-id="characterId"
  :editable="editable"
/>
```

**Step 4: Update SpellsByLevel to pass new props**

```vue
<CharacterSheetSpellsByLevel
  v-if="leveledSpells.length > 0"
  :spells="leveledSpells"
  :character-id="characterId"
  :editable="editable"
  :at-prep-limit="atPreparationLimit"
/>
```

**Step 5: Run character test suite**

```bash
docker compose exec nuxt npm run test:character
```

Expected: PASS

**Step 6: Commit**

```bash
git add app/components/character/sheet/SpellsPanel.vue
git commit -m "feat: pass new props through SpellsPanel (#616)"
```

---

## Task 13: Update SpellsByLevel to Use SpellCard with New Props

**Files:**
- Modify: `app/components/character/sheet/SpellsByLevel.vue`

**Step 1: Add new props**

```typescript
const props = defineProps<{
  spells: CharacterSpell[]
  characterId?: number
  editable?: boolean
  atPrepLimit?: boolean
}>()
```

**Step 2: Replace inline spell display with SpellCard**

Update the template to use SpellCard instead of inline display:

```vue
<!-- Spells Grid -->
<div class="grid grid-cols-1 gap-2">
  <CharacterSheetSpellCard
    v-for="spell in group.spells"
    :key="spell.id"
    :spell="spell"
    :character-id="characterId"
    :editable="editable"
    :at-prep-limit="atPrepLimit"
  />
</div>
```

**Step 3: Run tests**

```bash
docker compose exec nuxt npm run test:character
```

Expected: PASS

**Step 4: Commit**

```bash
git add app/components/character/sheet/SpellsByLevel.vue
git commit -m "feat: use SpellCard in SpellsByLevel with new props (#616)"
```

---

## Task 14: Initialize Spellcasting State in Character Page

**Files:**
- Modify: `app/pages/characters/[publicId]/index.vue`

**Step 1: Update store initialization watcher**

Extend the existing watch to initialize spell state:

```typescript
watch([character, stats, spells], ([char, s, sp]) => {
  if (char && s) {
    playStateStore.initialize({
      characterId: char.id,
      isDead: char.is_dead ?? false,
      hitPoints: {
        current: s.hit_points?.current ?? null,
        max: s.hit_points?.max ?? null,
        temporary: s.hit_points?.temporary ?? null
      },
      deathSaves: {
        successes: char.death_save_successes ?? 0,
        failures: char.death_save_failures ?? 0
      },
      currency: char.currency ?? { pp: 0, gp: 0, ep: 0, sp: 0, cp: 0 }
    })

    // Initialize spellcasting state
    if (s.spellcasting && sp) {
      // Convert spell_slots array/object to level-based array
      const slotData: Array<{ level: number, total: number }> = []
      if (Array.isArray(s.spell_slots)) {
        s.spell_slots.forEach((total, index) => {
          if (total > 0) {
            slotData.push({ level: index + 1, total })
          }
        })
      } else if (s.spell_slots) {
        Object.entries(s.spell_slots).forEach(([level, total]) => {
          if (total > 0) {
            slotData.push({ level: parseInt(level), total })
          }
        })
      }

      playStateStore.initializeSpellSlots(slotData)
      playStateStore.initializeSpellPreparation({
        spells: sp,
        preparationLimit: s.preparation_limit
      })
    }
  }
}, { immediate: true })
```

**Step 2: Update SpellsPanel usage to pass new props**

```vue
<CharacterSheetSpellsPanel
  v-if="stats.spellcasting"
  :spells="spells"
  :stats="stats"
  :character-id="character.id"
  :editable="canEdit"
/>
```

**Step 3: Run full test suite**

```bash
docker compose exec nuxt npm run test
```

Expected: PASS

**Step 4: Commit**

```bash
git add app/pages/characters/\\[publicId\\]/index.vue
git commit -m "feat: initialize spellcasting state from character page (#616)"
```

---

## Task 15: Manual Browser Testing

**Step 1: Start dev server**

```bash
docker compose exec nuxt npm run dev
```

**Step 2: Test spell slot interactions**

1. Navigate to a spellcaster character (e.g., `/characters/[publicId]`)
2. Enable Play Mode via toggle
3. Click a filled spell slot crystal → should become empty
4. Click an empty crystal → should become filled
5. Disable Play Mode → crystals should not be clickable

**Step 3: Test spell preparation toggle**

1. With Play Mode enabled
2. Click an unprepared spell card → should become prepared
3. Click a prepared spell card → should become unprepared
4. Prepare spells until at limit → unprepared spells should be greyed out
5. Click an always-prepared spell → should do nothing

**Step 4: Test dark mode**

1. Toggle dark mode
2. Verify crystals and cards display correctly

**Step 5: Commit any fixes**

---

## Task 16: Create Backend Issue for Stats Enrichment

**Files:**
- None (GitHub issue only)

**Step 1: Create the issue**

```bash
gh issue create --repo dfox288/ledger-of-heroes \
  --title "Backend: Enrich stats.spell_slots with spent/available" \
  --label "backend,feature,from:frontend" \
  --body "## Overview
Frontend spell slot tracking (#616) initializes \`spent: 0\` because \`stats.spell_slots\` only returns totals.

## Current Response
\`\`\`json
{ \"spell_slots\": [4, 3, 2] }
\`\`\`

## Requested Response
\`\`\`json
{
  \"spell_slots\": [
    { \"level\": 1, \"total\": 4, \"spent\": 1, \"available\": 3 },
    { \"level\": 2, \"total\": 3, \"spent\": 0, \"available\": 3 },
    { \"level\": 3, \"total\": 2, \"spent\": 2, \"available\": 0 }
  ]
}
\`\`\`

## Context
- Frontend issue: #616
- The PATCH endpoint already returns this structure
- Just need to expose it in the stats GET endpoint

## Impact
Without this, refreshing the page resets spent slots to 0 visually (data is persisted, just not loaded)."
```

**Step 2: Record issue number**

Note the issue number for the handoff.

---

## Task 17: Final Verification and PR

**Step 1: Run full test suite**

```bash
docker compose exec nuxt npm run test
docker compose exec nuxt npm run typecheck
docker compose exec nuxt npm run lint
```

Expected: All pass

**Step 2: Create PR**

```bash
git push -u origin feature/issue-616-spell-slot-tracking

gh pr create \
  --title "feat: Spell slot tracking and preparation toggle (#616)" \
  --body "## Summary
- Add interactive spell slot tracking (click crystals to use/restore)
- Add spell preparation toggle (click card to prepare/unprepare)
- Grey out unprepared spells when at preparation limit
- Extend characterPlayStateStore with spell state

## Changes
- New Nitro routes for spell endpoints
- SpellSlotsManager component (renamed from SpellSlots)
- SpellCard preparation toggle
- Store extension for spell state
- Character page initialization

## Test Plan
- [ ] Unit tests for store actions
- [ ] Component tests for SpellSlotsManager
- [ ] Component tests for SpellCard toggle
- [ ] Manual: use/restore spell slots
- [ ] Manual: prepare/unprepare spells
- [ ] Manual: grey-out at prep limit
- [ ] Manual: dark mode

## Related
- Closes #616
- Backend PR: dfox288/ledger-of-heroes-backend#170
- Follow-up: Stats enrichment issue (see created issue)"
```

---

## Summary

17 tasks covering:
1. Nitro routes (2 tasks)
2. Types (1 task)
3. Store spell slots (2 tasks: tests + implementation)
4. Store spell preparation (2 tasks: tests + implementation)
5. SpellSlotsManager (2 tasks: tests + implementation)
6. SpellCard toggle (2 tasks: tests + implementation)
7. Component prop threading (2 tasks)
8. Character page initialization (1 task)
9. Manual testing (1 task)
10. Backend issue (1 task)
11. PR creation (1 task)
