# Character Builder Phase 3: Ability Scores - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement ability score assignment step with three methods (Standard Array, Point Buy, Manual) and racial bonus display.

**Architecture:** Parent StepAbilities component with method selector delegates to three specialized input components. Each input component manages its own validation logic and emits scores via v-model. Store action saves method + scores to API.

**Tech Stack:** Vue 3, Pinia, NuxtUI 4, Vitest, TypeScript

**Design Document:** `docs/plans/2025-12-03-character-builder-phase3-abilities-design.md`

---

## Task 1: Add Store Action

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Test: `tests/stores/characterBuilder.test.ts`

**Step 1: Write the failing test**

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
describe('saveAbilityScores', () => {
  it('saves ability scores and method to API', async () => {
    const store = useCharacterBuilderStore()
    store.characterId = 1

    await store.saveAbilityScores('point_buy', {
      strength: 14,
      dexterity: 12,
      constitution: 15,
      intelligence: 10,
      wisdom: 13,
      charisma: 8
    })

    expect(store.abilityScores).toEqual({
      strength: 14,
      dexterity: 12,
      constitution: 15,
      intelligence: 10,
      wisdom: 13,
      charisma: 8
    })
    expect(store.abilityScoreMethod).toBe('point_buy')
  })

  it('sets error on API failure', async () => {
    const store = useCharacterBuilderStore()
    store.characterId = 999 // Will cause 404

    await expect(store.saveAbilityScores('manual', {
      strength: 10, dexterity: 10, constitution: 10,
      intelligence: 10, wisdom: 10, charisma: 10
    })).rejects.toThrow()

    expect(store.error).toBe('Failed to save ability scores')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose`

Expected: FAIL - `saveAbilityScores` not defined

**Step 3: Write minimal implementation**

Add to `app/stores/characterBuilder.ts`:

```typescript
// Add to state section
const abilityScoreMethod = ref<'standard_array' | 'point_buy' | 'manual'>('manual')

// Add to actions section
async function saveAbilityScores(
  method: 'standard_array' | 'point_buy' | 'manual',
  scores: AbilityScores
): Promise<void> {
  isLoading.value = true
  error.value = null

  try {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: {
        ability_score_method: method,
        strength: scores.strength,
        dexterity: scores.dexterity,
        constitution: scores.constitution,
        intelligence: scores.intelligence,
        wisdom: scores.wisdom,
        charisma: scores.charisma
      }
    })

    abilityScores.value = scores
    abilityScoreMethod.value = method
    await refreshStats()
  } catch (err: unknown) {
    error.value = 'Failed to save ability scores'
    throw err
  } finally {
    isLoading.value = false
  }
}

// Add to return statement
return {
  // ... existing exports
  abilityScoreMethod,
  saveAbilityScores,
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose`

Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat(character): add saveAbilityScores store action"
```

---

## Task 2: Create ManualInput Component

**Files:**
- Create: `app/components/character/builder/ManualInput.vue`
- Create: `tests/components/character/builder/ManualInput.test.ts`

**Step 1: Write the failing test**

Create `tests/components/character/builder/ManualInput.test.ts`:

```typescript
// tests/components/character/builder/ManualInput.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import ManualInput from '~/components/character/builder/ManualInput.vue'

const defaultScores = {
  strength: 10,
  dexterity: 10,
  constitution: 10,
  intelligence: 10,
  wisdom: 10,
  charisma: 10
}

describe('ManualInput', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders 6 ability score inputs', async () => {
    const wrapper = await mountSuspended(ManualInput, {
      props: { modelValue: defaultScores }
    })
    expect(wrapper.findAll('input[type="number"]')).toHaveLength(6)
  })

  it('displays current values', async () => {
    const scores = { ...defaultScores, strength: 15 }
    const wrapper = await mountSuspended(ManualInput, {
      props: { modelValue: scores }
    })
    const strInput = wrapper.find('[data-testid="input-strength"]')
    expect((strInput.element as HTMLInputElement).value).toBe('15')
  })

  it('emits update when value changes', async () => {
    const wrapper = await mountSuspended(ManualInput, {
      props: { modelValue: defaultScores }
    })
    const strInput = wrapper.find('[data-testid="input-strength"]')
    await strInput.setValue(14)

    const emitted = wrapper.emitted('update:modelValue')
    expect(emitted).toBeTruthy()
    expect(emitted![0][0]).toMatchObject({ strength: 14 })
  })

  it('reports valid when all scores in range 3-20', async () => {
    const wrapper = await mountSuspended(ManualInput, {
      props: { modelValue: defaultScores }
    })
    const emittedValid = wrapper.emitted('update:valid')
    expect(emittedValid).toBeTruthy()
    expect(emittedValid![0][0]).toBe(true)
  })

  it('reports invalid when score out of range', async () => {
    const invalidScores = { ...defaultScores, strength: 25 }
    const wrapper = await mountSuspended(ManualInput, {
      props: { modelValue: invalidScores }
    })
    const emittedValid = wrapper.emitted('update:valid')
    expect(emittedValid).toBeTruthy()
    expect(emittedValid![0][0]).toBe(false)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/ManualInput.test.ts --reporter=verbose`

Expected: FAIL - component not found

**Step 3: Write minimal implementation**

Create `app/components/character/builder/ManualInput.vue`:

```vue
<!-- app/components/character/builder/ManualInput.vue -->
<script setup lang="ts">
import type { AbilityScores } from '~/types'

const props = defineProps<{
  modelValue: AbilityScores
}>()

const emit = defineEmits<{
  'update:modelValue': [scores: AbilityScores]
  'update:valid': [valid: boolean]
}>()

const abilities = [
  { key: 'strength', label: 'STR', name: 'Strength' },
  { key: 'dexterity', label: 'DEX', name: 'Dexterity' },
  { key: 'constitution', label: 'CON', name: 'Constitution' },
  { key: 'intelligence', label: 'INT', name: 'Intelligence' },
  { key: 'wisdom', label: 'WIS', name: 'Wisdom' },
  { key: 'charisma', label: 'CHA', name: 'Charisma' }
] as const

function updateScore(key: keyof AbilityScores, value: number) {
  const newScores = { ...props.modelValue, [key]: value }
  emit('update:modelValue', newScores)
}

const isValid = computed(() => {
  return Object.values(props.modelValue).every(
    score => score >= 3 && score <= 20
  )
})

// Emit validity on mount and when it changes
watch(isValid, (valid) => emit('update:valid', valid), { immediate: true })
</script>

<template>
  <div class="space-y-4">
    <p class="text-sm text-gray-600 dark:text-gray-400">
      Enter your ability scores (3-20 range)
    </p>
    <div class="grid grid-cols-2 md:grid-cols-3 gap-4">
      <div
        v-for="ability in abilities"
        :key="ability.key"
        class="flex flex-col"
      >
        <label
          :for="`ability-${ability.key}`"
          class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
        >
          {{ ability.name }}
        </label>
        <UInput
          :id="`ability-${ability.key}`"
          type="number"
          :model-value="modelValue[ability.key]"
          :min="3"
          :max="20"
          :data-testid="`input-${ability.key}`"
          @update:model-value="updateScore(ability.key, Number($event))"
        />
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/ManualInput.test.ts --reporter=verbose`

Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/ManualInput.vue tests/components/character/builder/ManualInput.test.ts
git commit -m "feat(character): add ManualInput component for ability scores"
```

---

## Task 3: Create StandardArrayInput Component

**Files:**
- Create: `app/components/character/builder/StandardArrayInput.vue`
- Create: `tests/components/character/builder/StandardArrayInput.test.ts`

**Step 1: Write the failing test**

Create `tests/components/character/builder/StandardArrayInput.test.ts`:

```typescript
// tests/components/character/builder/StandardArrayInput.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StandardArrayInput from '~/components/character/builder/StandardArrayInput.vue'

const emptyScores = {
  strength: null,
  dexterity: null,
  constitution: null,
  intelligence: null,
  wisdom: null,
  charisma: null
}

describe('StandardArrayInput', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders 6 ability score selects', async () => {
    const wrapper = await mountSuspended(StandardArrayInput, {
      props: { modelValue: emptyScores }
    })
    expect(wrapper.findAll('[data-testid^="select-"]')).toHaveLength(6)
  })

  it('shows standard array values as options', async () => {
    const wrapper = await mountSuspended(StandardArrayInput, {
      props: { modelValue: emptyScores }
    })
    const text = wrapper.text()
    expect(text).toContain('15')
    expect(text).toContain('14')
    expect(text).toContain('13')
    expect(text).toContain('12')
    expect(text).toContain('10')
    expect(text).toContain('8')
  })

  it('emits update when value selected', async () => {
    const wrapper = await mountSuspended(StandardArrayInput, {
      props: { modelValue: emptyScores }
    })

    // Find and interact with STR select
    const strSelect = wrapper.find('[data-testid="select-strength"]')
    await strSelect.trigger('click')

    const emitted = wrapper.emitted('update:modelValue')
    // Component should emit on interaction
    expect(emitted).toBeDefined()
  })

  it('reports invalid when not all scores assigned', async () => {
    const wrapper = await mountSuspended(StandardArrayInput, {
      props: { modelValue: emptyScores }
    })
    const emittedValid = wrapper.emitted('update:valid')
    expect(emittedValid).toBeTruthy()
    expect(emittedValid![0][0]).toBe(false)
  })

  it('reports valid when all scores assigned', async () => {
    const fullScores = {
      strength: 15,
      dexterity: 14,
      constitution: 13,
      intelligence: 12,
      wisdom: 10,
      charisma: 8
    }
    const wrapper = await mountSuspended(StandardArrayInput, {
      props: { modelValue: fullScores }
    })
    const emittedValid = wrapper.emitted('update:valid')
    expect(emittedValid).toBeTruthy()
    expect(emittedValid![0][0]).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StandardArrayInput.test.ts --reporter=verbose`

Expected: FAIL - component not found

**Step 3: Write minimal implementation**

Create `app/components/character/builder/StandardArrayInput.vue`:

```vue
<!-- app/components/character/builder/StandardArrayInput.vue -->
<script setup lang="ts">
type NullableScores = {
  strength: number | null
  dexterity: number | null
  constitution: number | null
  intelligence: number | null
  wisdom: number | null
  charisma: number | null
}

const props = defineProps<{
  modelValue: NullableScores
}>()

const emit = defineEmits<{
  'update:modelValue': [scores: NullableScores]
  'update:valid': [valid: boolean]
}>()

const STANDARD_ARRAY = [15, 14, 13, 12, 10, 8] as const

const abilities = [
  { key: 'strength', label: 'STR', name: 'Strength' },
  { key: 'dexterity', label: 'DEX', name: 'Dexterity' },
  { key: 'constitution', label: 'CON', name: 'Constitution' },
  { key: 'intelligence', label: 'INT', name: 'Intelligence' },
  { key: 'wisdom', label: 'WIS', name: 'Wisdom' },
  { key: 'charisma', label: 'CHA', name: 'Charisma' }
] as const

type AbilityKey = typeof abilities[number]['key']

/**
 * Get available values for a specific ability
 * Excludes values already assigned to other abilities
 */
function getAvailableValues(currentKey: AbilityKey): number[] {
  const usedValues = new Set<number>()

  for (const ability of abilities) {
    if (ability.key !== currentKey) {
      const value = props.modelValue[ability.key]
      if (value !== null) {
        usedValues.add(value)
      }
    }
  }

  return STANDARD_ARRAY.filter(v => !usedValues.has(v))
}

function updateScore(key: AbilityKey, value: number | null) {
  const newScores = { ...props.modelValue, [key]: value }
  emit('update:modelValue', newScores)
}

const isValid = computed(() => {
  const values = Object.values(props.modelValue)
  // All must be assigned
  if (values.some(v => v === null)) return false
  // All must be from standard array
  const validValues = new Set(STANDARD_ARRAY)
  return values.every(v => validValues.has(v as number))
})

// Emit validity on mount and when it changes
watch(isValid, (valid) => emit('update:valid', valid), { immediate: true })
</script>

<template>
  <div class="space-y-4">
    <p class="text-sm text-gray-600 dark:text-gray-400">
      Assign each value once: 15, 14, 13, 12, 10, 8
    </p>
    <div class="grid grid-cols-2 md:grid-cols-3 gap-4">
      <div
        v-for="ability in abilities"
        :key="ability.key"
        class="flex flex-col"
      >
        <label
          :for="`ability-${ability.key}`"
          class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-1"
        >
          {{ ability.name }}
        </label>
        <USelectMenu
          :id="`ability-${ability.key}`"
          :model-value="modelValue[ability.key]"
          :items="getAvailableValues(ability.key).map(v => ({ label: String(v), value: v }))"
          value-key="value"
          placeholder="Select"
          :data-testid="`select-${ability.key}`"
          @update:model-value="updateScore(ability.key, $event)"
        />
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StandardArrayInput.test.ts --reporter=verbose`

Expected: PASS (or adjust for USelectMenu behavior)

**Step 5: Commit**

```bash
git add app/components/character/builder/StandardArrayInput.vue tests/components/character/builder/StandardArrayInput.test.ts
git commit -m "feat(character): add StandardArrayInput component for ability scores"
```

---

## Task 4: Create PointBuyInput Component

**Files:**
- Create: `app/components/character/builder/PointBuyInput.vue`
- Create: `tests/components/character/builder/PointBuyInput.test.ts`

**Step 1: Write the failing test**

Create `tests/components/character/builder/PointBuyInput.test.ts`:

```typescript
// tests/components/character/builder/PointBuyInput.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import PointBuyInput from '~/components/character/builder/PointBuyInput.vue'

const baseScores = {
  strength: 8,
  dexterity: 8,
  constitution: 8,
  intelligence: 8,
  wisdom: 8,
  charisma: 8
}

describe('PointBuyInput', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders 6 ability scores with +/- buttons', async () => {
    const wrapper = await mountSuspended(PointBuyInput, {
      props: { modelValue: baseScores }
    })
    expect(wrapper.findAll('[data-testid^="score-"]')).toHaveLength(6)
    expect(wrapper.findAll('[data-testid$="-increase"]')).toHaveLength(6)
    expect(wrapper.findAll('[data-testid$="-decrease"]')).toHaveLength(6)
  })

  it('shows points remaining', async () => {
    const wrapper = await mountSuspended(PointBuyInput, {
      props: { modelValue: baseScores }
    })
    // All 8s = 0 points spent, 27 remaining
    expect(wrapper.text()).toContain('27')
  })

  it('calculates points spent correctly', async () => {
    const spentScores = {
      strength: 15, // 9 points
      dexterity: 14, // 7 points
      constitution: 13, // 5 points
      intelligence: 8, // 0 points
      wisdom: 8, // 0 points
      charisma: 8 // 0 points
    } // Total: 21 points spent, 6 remaining

    const wrapper = await mountSuspended(PointBuyInput, {
      props: { modelValue: spentScores }
    })
    expect(wrapper.text()).toContain('6')
  })

  it('emits update when + clicked', async () => {
    const wrapper = await mountSuspended(PointBuyInput, {
      props: { modelValue: baseScores }
    })
    await wrapper.find('[data-testid="strength-increase"]').trigger('click')

    const emitted = wrapper.emitted('update:modelValue')
    expect(emitted).toBeTruthy()
    expect(emitted![0][0]).toMatchObject({ strength: 9 })
  })

  it('emits update when - clicked', async () => {
    const scores = { ...baseScores, strength: 10 }
    const wrapper = await mountSuspended(PointBuyInput, {
      props: { modelValue: scores }
    })
    await wrapper.find('[data-testid="strength-decrease"]').trigger('click')

    const emitted = wrapper.emitted('update:modelValue')
    expect(emitted).toBeTruthy()
    expect(emitted![0][0]).toMatchObject({ strength: 9 })
  })

  it('disables - button at minimum (8)', async () => {
    const wrapper = await mountSuspended(PointBuyInput, {
      props: { modelValue: baseScores }
    })
    const decreaseBtn = wrapper.find('[data-testid="strength-decrease"]')
    expect(decreaseBtn.attributes('disabled')).toBeDefined()
  })

  it('disables + button at maximum (15)', async () => {
    const maxScores = { ...baseScores, strength: 15 }
    const wrapper = await mountSuspended(PointBuyInput, {
      props: { modelValue: maxScores }
    })
    const increaseBtn = wrapper.find('[data-testid="strength-increase"]')
    expect(increaseBtn.attributes('disabled')).toBeDefined()
  })

  it('reports valid when points not exceeded', async () => {
    const wrapper = await mountSuspended(PointBuyInput, {
      props: { modelValue: baseScores }
    })
    const emittedValid = wrapper.emitted('update:valid')
    expect(emittedValid).toBeTruthy()
    expect(emittedValid![0][0]).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/PointBuyInput.test.ts --reporter=verbose`

Expected: FAIL - component not found

**Step 3: Write minimal implementation**

Create `app/components/character/builder/PointBuyInput.vue`:

```vue
<!-- app/components/character/builder/PointBuyInput.vue -->
<script setup lang="ts">
import type { AbilityScores } from '~/types'

const props = defineProps<{
  modelValue: AbilityScores
}>()

const emit = defineEmits<{
  'update:modelValue': [scores: AbilityScores]
  'update:valid': [valid: boolean]
}>()

const TOTAL_POINTS = 27
const MIN_SCORE = 8
const MAX_SCORE = 15

// Point cost for each score value
const POINT_COSTS: Record<number, number> = {
  8: 0, 9: 1, 10: 2, 11: 3, 12: 4, 13: 5, 14: 7, 15: 9
}

const abilities = [
  { key: 'strength', label: 'STR', name: 'Strength' },
  { key: 'dexterity', label: 'DEX', name: 'Dexterity' },
  { key: 'constitution', label: 'CON', name: 'Constitution' },
  { key: 'intelligence', label: 'INT', name: 'Intelligence' },
  { key: 'wisdom', label: 'WIS', name: 'Wisdom' },
  { key: 'charisma', label: 'CHA', name: 'Charisma' }
] as const

type AbilityKey = typeof abilities[number]['key']

const pointsSpent = computed(() => {
  return Object.values(props.modelValue).reduce(
    (sum, score) => sum + (POINT_COSTS[score] ?? 0),
    0
  )
})

const pointsRemaining = computed(() => TOTAL_POINTS - pointsSpent.value)

function getCostForNextIncrease(score: number): number {
  if (score >= MAX_SCORE) return 0
  return POINT_COSTS[score + 1] - POINT_COSTS[score]
}

function canIncrease(key: AbilityKey): boolean {
  const score = props.modelValue[key]
  if (score >= MAX_SCORE) return false
  const cost = getCostForNextIncrease(score)
  return pointsRemaining.value >= cost
}

function canDecrease(key: AbilityKey): boolean {
  return props.modelValue[key] > MIN_SCORE
}

function increase(key: AbilityKey) {
  if (!canIncrease(key)) return
  const newScores = { ...props.modelValue, [key]: props.modelValue[key] + 1 }
  emit('update:modelValue', newScores)
}

function decrease(key: AbilityKey) {
  if (!canDecrease(key)) return
  const newScores = { ...props.modelValue, [key]: props.modelValue[key] - 1 }
  emit('update:modelValue', newScores)
}

const isValid = computed(() => {
  // Valid if all scores in range and points not exceeded
  const scores = Object.values(props.modelValue)
  const allInRange = scores.every(s => s >= MIN_SCORE && s <= MAX_SCORE)
  return allInRange && pointsSpent.value <= TOTAL_POINTS
})

// Emit validity on mount and when it changes
watch(isValid, (valid) => emit('update:valid', valid), { immediate: true })
</script>

<template>
  <div class="space-y-4">
    <div class="flex items-center justify-between">
      <p class="text-sm text-gray-600 dark:text-gray-400">
        Spend points to increase scores (8-15 range)
      </p>
      <div class="text-lg font-semibold">
        Points: <span :class="pointsRemaining < 0 ? 'text-red-500' : 'text-green-600'">{{ pointsRemaining }}</span> / {{ TOTAL_POINTS }}
      </div>
    </div>

    <div class="grid grid-cols-2 md:grid-cols-3 gap-4">
      <div
        v-for="ability in abilities"
        :key="ability.key"
        class="flex flex-col items-center p-3 bg-gray-50 dark:bg-gray-800 rounded-lg"
      >
        <span class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
          {{ ability.name }}
        </span>
        <div class="flex items-center gap-2">
          <UButton
            :data-testid="`${ability.key}-decrease`"
            icon="i-heroicons-minus"
            size="sm"
            variant="soft"
            :disabled="!canDecrease(ability.key)"
            @click="decrease(ability.key)"
          />
          <span
            :data-testid="`score-${ability.key}`"
            class="w-8 text-center text-xl font-bold"
          >
            {{ modelValue[ability.key] }}
          </span>
          <UButton
            :data-testid="`${ability.key}-increase`"
            icon="i-heroicons-plus"
            size="sm"
            variant="soft"
            :disabled="!canIncrease(ability.key)"
            @click="increase(ability.key)"
          />
        </div>
        <span class="text-xs text-gray-500 mt-1">
          Cost: {{ POINT_COSTS[modelValue[ability.key]] }}
        </span>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/PointBuyInput.test.ts --reporter=verbose`

Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/PointBuyInput.vue tests/components/character/builder/PointBuyInput.test.ts
git commit -m "feat(character): add PointBuyInput component for ability scores"
```

---

## Task 5: Update StepAbilities Component

**Files:**
- Modify: `app/components/character/builder/StepAbilities.vue`
- Create: `tests/components/character/builder/StepAbilities.test.ts`

**Step 1: Write the failing test**

Create `tests/components/character/builder/StepAbilities.test.ts`:

```typescript
// tests/components/character/builder/StepAbilities.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepAbilities from '~/components/character/builder/StepAbilities.vue'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

describe('StepAbilities', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders method selector with 3 options', async () => {
    const wrapper = await mountSuspended(StepAbilities)
    expect(wrapper.text()).toContain('Standard Array')
    expect(wrapper.text()).toContain('Point Buy')
    expect(wrapper.text()).toContain('Manual')
  })

  it('shows ManualInput by default', async () => {
    const wrapper = await mountSuspended(StepAbilities)
    expect(wrapper.findComponent({ name: 'ManualInput' }).exists()).toBe(true)
  })

  it('shows racial bonuses when race selected', async () => {
    const store = useCharacterBuilderStore()
    store.selectedRace = {
      id: 1,
      name: 'Dwarf',
      modifiers: [
        { id: 1, modifier_category: 'ability_score', value: '2', ability_score: { code: 'CON', name: 'Constitution' } }
      ]
    } as any

    const wrapper = await mountSuspended(StepAbilities)
    expect(wrapper.text()).toContain('Racial Bonuses')
    expect(wrapper.text()).toContain('Constitution')
    expect(wrapper.text()).toContain('+2')
  })

  it('disables save button when invalid', async () => {
    const wrapper = await mountSuspended(StepAbilities)
    // Find save button - initially might be disabled depending on default state
    const saveBtn = wrapper.find('[data-testid="save-abilities"]')
    expect(saveBtn.exists()).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepAbilities.test.ts --reporter=verbose`

Expected: FAIL - component doesn't have expected content

**Step 3: Write minimal implementation**

Replace `app/components/character/builder/StepAbilities.vue`:

```vue
<!-- app/components/character/builder/StepAbilities.vue -->
<script setup lang="ts">
import type { AbilityScores } from '~/types'

const store = useCharacterBuilderStore()
const { selectedRace, abilityScores, abilityScoreMethod, isLoading, error } = storeToRefs(store)

type Method = 'standard_array' | 'point_buy' | 'manual'

const selectedMethod = ref<Method>(abilityScoreMethod.value || 'manual')

// Local scores for editing (converted for standard array which uses nullable)
const localScores = ref<AbilityScores>({ ...abilityScores.value })

// For standard array, we need nullable scores
const nullableScores = ref<Record<keyof AbilityScores, number | null>>({
  strength: null,
  dexterity: null,
  constitution: null,
  intelligence: null,
  wisdom: null,
  charisma: null
})

// Track validity from child components
const isInputValid = ref(false)

// Racial ability bonuses
const racialBonuses = computed(() => {
  if (!selectedRace.value?.modifiers) return []
  return selectedRace.value.modifiers.filter(
    m => m.modifier_category === 'ability_score' && m.ability_score
  )
})

// Calculate final scores with racial bonuses
const finalScores = computed(() => {
  const base = selectedMethod.value === 'standard_array'
    ? nullableScores.value
    : localScores.value

  const result: Record<string, { base: number | null, bonus: number, total: number | null }> = {}

  const abilities = ['strength', 'dexterity', 'constitution', 'intelligence', 'wisdom', 'charisma'] as const
  const codeMap: Record<string, string> = {
    STR: 'strength', DEX: 'dexterity', CON: 'constitution',
    INT: 'intelligence', WIS: 'wisdom', CHA: 'charisma'
  }

  for (const ability of abilities) {
    const baseScore = base[ability]
    const bonus = racialBonuses.value
      .filter(m => codeMap[m.ability_score?.code ?? ''] === ability)
      .reduce((sum, m) => sum + Number(m.value), 0)

    result[ability] = {
      base: baseScore,
      bonus,
      total: baseScore !== null ? baseScore + bonus : null
    }
  }

  return result
})

const canSave = computed(() => {
  if (isLoading.value) return false
  return isInputValid.value
})

async function saveAndContinue() {
  if (!canSave.value) return

  const scores = selectedMethod.value === 'standard_array'
    ? Object.fromEntries(
        Object.entries(nullableScores.value).map(([k, v]) => [k, v ?? 10])
      ) as AbilityScores
    : localScores.value

  await store.saveAbilityScores(selectedMethod.value, scores)
  store.nextStep()
}

// Method options for selector
const methodOptions = [
  { label: 'Standard Array', value: 'standard_array' },
  { label: 'Point Buy', value: 'point_buy' },
  { label: 'Manual', value: 'manual' }
]

// Reset scores when method changes
watch(selectedMethod, (newMethod) => {
  if (newMethod === 'standard_array') {
    // Reset to unassigned
    nullableScores.value = {
      strength: null, dexterity: null, constitution: null,
      intelligence: null, wisdom: null, charisma: null
    }
  } else if (newMethod === 'point_buy') {
    // Reset to base 8s
    localScores.value = {
      strength: 8, dexterity: 8, constitution: 8,
      intelligence: 8, wisdom: 8, charisma: 8
    }
  } else {
    // Reset to 10s for manual
    localScores.value = {
      strength: 10, dexterity: 10, constitution: 10,
      intelligence: 10, wisdom: 10, charisma: 10
    }
  }
})
</script>

<template>
  <div class="space-y-6">
    <div>
      <h2 class="text-xl font-semibold mb-2">
        Assign Ability Scores
      </h2>
      <p class="text-gray-600 dark:text-gray-400">
        Choose a method and assign your character's six ability scores.
      </p>
    </div>

    <!-- Method Selector -->
    <div>
      <label class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
        Method
      </label>
      <UButtonGroup>
        <UButton
          v-for="option in methodOptions"
          :key="option.value"
          :variant="selectedMethod === option.value ? 'solid' : 'outline'"
          @click="selectedMethod = option.value as Method"
        >
          {{ option.label }}
        </UButton>
      </UButtonGroup>
    </div>

    <!-- Method-specific Input -->
    <div class="mt-6">
      <CharacterBuilderStandardArrayInput
        v-if="selectedMethod === 'standard_array'"
        v-model="nullableScores"
        @update:valid="isInputValid = $event"
      />
      <CharacterBuilderPointBuyInput
        v-else-if="selectedMethod === 'point_buy'"
        v-model="localScores"
        @update:valid="isInputValid = $event"
      />
      <CharacterBuilderManualInput
        v-else
        v-model="localScores"
        @update:valid="isInputValid = $event"
      />
    </div>

    <!-- Racial Bonuses -->
    <div
      v-if="racialBonuses.length > 0"
      class="p-4 bg-race-50 dark:bg-race-900/20 rounded-lg"
    >
      <h3 class="font-semibold text-gray-900 dark:text-gray-100 mb-2">
        Racial Bonuses ({{ selectedRace?.name }})
      </h3>
      <div class="flex flex-wrap gap-2">
        <UBadge
          v-for="bonus in racialBonuses"
          :key="bonus.id"
          color="race"
          variant="subtle"
          size="md"
        >
          {{ bonus.ability_score?.name }} +{{ bonus.value }}
        </UBadge>
      </div>
    </div>

    <!-- Final Scores Summary -->
    <div class="p-4 bg-gray-50 dark:bg-gray-800 rounded-lg">
      <h3 class="font-semibold text-gray-900 dark:text-gray-100 mb-3">
        Final Ability Scores
      </h3>
      <div class="grid grid-cols-3 md:grid-cols-6 gap-4 text-center">
        <div
          v-for="(data, ability) in finalScores"
          :key="ability"
          class="p-2"
        >
          <div class="text-xs font-medium text-gray-500 uppercase">
            {{ ability.slice(0, 3) }}
          </div>
          <div class="text-lg font-bold">
            <template v-if="data.total !== null">
              {{ data.total }}
              <span
                v-if="data.bonus > 0"
                class="text-xs text-race-600 dark:text-race-400"
              >
                ({{ data.base }}+{{ data.bonus }})
              </span>
            </template>
            <span
              v-else
              class="text-gray-400"
            >—</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Error Display -->
    <UAlert
      v-if="error"
      color="error"
      icon="i-heroicons-exclamation-triangle"
      :title="error"
    />

    <!-- Save Button -->
    <div class="flex justify-end">
      <UButton
        data-testid="save-abilities"
        :disabled="!canSave"
        :loading="isLoading"
        @click="saveAndContinue"
      >
        Save & Continue
      </UButton>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepAbilities.test.ts --reporter=verbose`

Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/StepAbilities.vue tests/components/character/builder/StepAbilities.test.ts
git commit -m "feat(character): implement StepAbilities with method selector and racial bonuses"
```

---

## Task 6: Integration Test & Final Verification

**Step 1: Run all character builder tests**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder tests/stores/characterBuilder.test.ts --reporter=verbose`

Expected: All tests PASS

**Step 2: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No errors

**Step 3: Run lint**

Run: `docker compose exec nuxt npm run lint:fix`

Expected: No errors or auto-fixed

**Step 4: Manual browser test**

1. Navigate to `/characters/create`
2. Complete Name, Race, Class steps
3. On Ability Scores step:
   - Test Standard Array (assign all 6 values)
   - Test Point Buy (spend exactly 27 points)
   - Test Manual (enter values 3-20)
4. Verify racial bonuses display correctly
5. Save and verify character updates

**Step 5: Final commit**

```bash
git add -A
git commit -m "test(character): verify Phase 3 ability scores integration"
```

---

## Summary

| Task | Component | Tests |
|------|-----------|-------|
| 1 | Store: `saveAbilityScores` | 2 tests |
| 2 | `ManualInput.vue` | 5 tests |
| 3 | `StandardArrayInput.vue` | 5 tests |
| 4 | `PointBuyInput.vue` | 9 tests |
| 5 | `StepAbilities.vue` | 4 tests |
| 6 | Integration verification | Manual |

**Total new tests:** ~25
**Estimated time:** 2-3 hours
