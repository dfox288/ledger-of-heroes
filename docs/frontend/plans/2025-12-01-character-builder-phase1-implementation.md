# Character Builder Phase 1: Foundation - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build the Pinia store, wizard container page, stepper component, and first step (Name) to create draft characters.

**Architecture:** Single-page wizard using `UStepper`, Pinia store for wizard state, API calls on step completion. Character is created as draft on step 1, updated as user progresses.

**Tech Stack:** Nuxt 4, NuxtUI 4, Pinia, Vitest, TypeScript

**Design Doc:** `docs/plans/2025-12-01-character-builder-frontend-design.md`

**GitHub Issue:** #89

---

## Pre-flight Checklist

- [ ] Branch: `feature/issue-89-character-builder`
- [ ] Docker running: `docker compose ps` shows nuxt container
- [ ] Backend running: `curl http://localhost:8080/api/v1/characters` returns data
- [ ] Tests passing: `docker compose exec nuxt npm run test:core` passes

---

## Task 1: Create TypeScript Types for Character Builder

**Files:**
- Create: `app/types/character.ts`

**Step 1.1: Create the types file**

```typescript
// app/types/character.ts

/**
 * Ability scores for character creation
 */
export interface AbilityScores {
  strength: number
  dexterity: number
  constitution: number
  intelligence: number
  wisdom: number
  charisma: number
}

/**
 * Ability score codes used by the API
 */
export type AbilityScoreCode = 'STR' | 'DEX' | 'CON' | 'INT' | 'WIS' | 'CHA'

/**
 * API response format for ability scores
 */
export interface AbilityScoresResponse {
  STR: number | null
  DEX: number | null
  CON: number | null
  INT: number | null
  WIS: number | null
  CHA: number | null
}

/**
 * Validation status from Character API
 */
export interface CharacterValidationStatus {
  is_complete: boolean
  missing: string[]
}

/**
 * Character summary (list view)
 */
export interface CharacterSummary {
  id: number
  name: string
  level: number
  is_complete: boolean
  race: { id: number; name: string; slug: string } | null
  class: { id: number; name: string; slug: string } | null
  background: { id: number; name: string; slug: string } | null
}

/**
 * Full character data from API
 */
export interface Character {
  id: number
  name: string
  level: number
  experience_points: number
  is_complete: boolean
  validation_status: CharacterValidationStatus
  ability_scores: AbilityScoresResponse
  modifiers: AbilityScoresResponse
  proficiency_bonus: number
  max_hit_points: number | null
  current_hit_points: number | null
  temp_hit_points: number
  armor_class: number | null
  race: { id: number; name: string; slug: string } | null
  class: { id: number; name: string; slug: string } | null
  background: { id: number; name: string; slug: string } | null
  created_at: string
  updated_at: string
}

/**
 * Character stats from /characters/{id}/stats endpoint
 */
export interface CharacterStats {
  character_id: number
  level: number
  proficiency_bonus: number
  ability_scores: Record<AbilityScoreCode, { score: number; modifier: number }>
  saving_throws: Record<AbilityScoreCode, { modifier: number; proficient: boolean }>
  armor_class: number | null
  hit_points: {
    max: number | null
    current: number | null
    temporary: number
  }
  spellcasting: {
    ability: AbilityScoreCode
    save_dc: number
    attack_bonus: number
  } | null
  spell_slots: Record<string, number>
  preparation_limit: number | null
  prepared_spell_count: number
}

/**
 * Wizard step definition
 */
export interface WizardStep {
  id: number
  name: string
  label: string
  icon: string
  isComplete: boolean
  isActive: boolean
  isDisabled: boolean
}
```

**Step 1.2: Export from types index**

Add to `app/types/index.ts`:

```typescript
export * from './character'
```

**Step 1.3: Verify types compile**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No new errors

**Step 1.4: Commit**

```bash
git add app/types/character.ts app/types/index.ts
git commit -m "feat(character): add TypeScript types for character builder

- AbilityScores interface for form state
- Character and CharacterStats for API responses
- CharacterValidationStatus for completion tracking
- WizardStep for stepper UI"
```

---

## Task 2: Create Character Builder Pinia Store

**Files:**
- Create: `app/stores/characterBuilder.ts`
- Create: `tests/stores/characterBuilder.test.ts`

**Step 2.1: Write failing tests for store initial state**

```typescript
// tests/stores/characterBuilder.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

describe('useCharacterBuilderStore', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  describe('initial state', () => {
    it('starts at step 1', () => {
      const store = useCharacterBuilderStore()
      expect(store.currentStep).toBe(1)
    })

    it('has no character ID initially', () => {
      const store = useCharacterBuilderStore()
      expect(store.characterId).toBeNull()
    })

    it('has empty name initially', () => {
      const store = useCharacterBuilderStore()
      expect(store.name).toBe('')
    })

    it('has default ability scores of 10', () => {
      const store = useCharacterBuilderStore()
      expect(store.abilityScores).toEqual({
        strength: 10,
        dexterity: 10,
        constitution: 10,
        intelligence: 10,
        wisdom: 10,
        charisma: 10
      })
    })

    it('calculates totalSteps as 6 for non-casters', () => {
      const store = useCharacterBuilderStore()
      expect(store.totalSteps).toBe(6)
    })

    it('isFirstStep is true at step 1', () => {
      const store = useCharacterBuilderStore()
      expect(store.isFirstStep).toBe(true)
    })

    it('isLastStep is false at step 1', () => {
      const store = useCharacterBuilderStore()
      expect(store.isLastStep).toBe(false)
    })
  })
})
```

**Step 2.2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts`

Expected: FAIL - Cannot find module '~/stores/characterBuilder'

**Step 2.3: Create minimal store to pass initial state tests**

```typescript
// app/stores/characterBuilder.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'
import type { AbilityScores, Character, CharacterStats, Race, CharacterClass, Background, CharacterSpell } from '~/types'

/**
 * Character Builder Wizard Store
 *
 * Manages state for the multi-step character creation wizard.
 * Character is created as draft on step 1, updated as user progresses.
 */
export const useCharacterBuilderStore = defineStore('characterBuilder', () => {
  // ══════════════════════════════════════════════════════════════
  // WIZARD NAVIGATION
  // ══════════════════════════════════════════════════════════════
  const currentStep = ref(1)
  const totalSteps = computed(() => isCaster.value ? 7 : 6)
  const isFirstStep = computed(() => currentStep.value === 1)
  const isLastStep = computed(() => currentStep.value === totalSteps.value)

  // ══════════════════════════════════════════════════════════════
  // CHARACTER DATA (mirrors API fields)
  // ══════════════════════════════════════════════════════════════
  const characterId = ref<number | null>(null)
  const name = ref('')
  const raceId = ref<number | null>(null)
  const subraceId = ref<number | null>(null)
  const classId = ref<number | null>(null)
  const backgroundId = ref<number | null>(null)
  const abilityScores = ref<AbilityScores>({
    strength: 10,
    dexterity: 10,
    constitution: 10,
    intelligence: 10,
    wisdom: 10,
    charisma: 10
  })

  // ══════════════════════════════════════════════════════════════
  // FETCHED REFERENCE DATA (for display without re-fetching)
  // ══════════════════════════════════════════════════════════════
  const selectedRace = ref<Race | null>(null)
  const selectedClass = ref<CharacterClass | null>(null)
  const selectedBackground = ref<Background | null>(null)
  const selectedSpells = ref<CharacterSpell[]>([])

  // ══════════════════════════════════════════════════════════════
  // COMPUTED STATS (from API)
  // ══════════════════════════════════════════════════════════════
  const characterData = ref<Character | null>(null)
  const characterStats = ref<CharacterStats | null>(null)

  // ══════════════════════════════════════════════════════════════
  // DERIVED STATE
  // ══════════════════════════════════════════════════════════════
  const isCaster = computed(() =>
    selectedClass.value?.spellcasting_ability !== null &&
    selectedClass.value?.spellcasting_ability !== undefined
  )

  const validationStatus = computed(() =>
    characterData.value?.validation_status ?? { is_complete: false, missing: [] }
  )

  const isComplete = computed(() =>
    validationStatus.value.is_complete
  )

  const racialBonuses = computed(() =>
    selectedRace.value?.modifiers?.filter(m => m.modifier_category === 'ability_score') ?? []
  )

  // ══════════════════════════════════════════════════════════════
  // LOADING STATE
  // ══════════════════════════════════════════════════════════════
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  // ══════════════════════════════════════════════════════════════
  // NAVIGATION ACTIONS
  // ══════════════════════════════════════════════════════════════
  function nextStep(): void {
    if (currentStep.value < totalSteps.value) {
      currentStep.value++
    }
  }

  function previousStep(): void {
    if (currentStep.value > 1) {
      currentStep.value--
    }
  }

  function goToStep(step: number): void {
    if (step >= 1 && step <= totalSteps.value) {
      currentStep.value = step
    }
  }

  // ══════════════════════════════════════════════════════════════
  // RESET ACTION
  // ══════════════════════════════════════════════════════════════
  function reset(): void {
    characterId.value = null
    name.value = ''
    raceId.value = null
    subraceId.value = null
    classId.value = null
    backgroundId.value = null
    abilityScores.value = {
      strength: 10,
      dexterity: 10,
      constitution: 10,
      intelligence: 10,
      wisdom: 10,
      charisma: 10
    }
    selectedRace.value = null
    selectedClass.value = null
    selectedBackground.value = null
    selectedSpells.value = []
    characterData.value = null
    characterStats.value = null
    currentStep.value = 1
    isLoading.value = false
    error.value = null
  }

  return {
    // State
    currentStep,
    totalSteps,
    isFirstStep,
    isLastStep,
    characterId,
    name,
    raceId,
    subraceId,
    classId,
    backgroundId,
    abilityScores,
    selectedRace,
    selectedClass,
    selectedBackground,
    selectedSpells,
    characterData,
    characterStats,
    isCaster,
    validationStatus,
    isComplete,
    racialBonuses,
    isLoading,
    error,
    // Actions
    nextStep,
    previousStep,
    goToStep,
    reset
  }
})
```

**Step 2.4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts`

Expected: All 7 tests PASS

**Step 2.5: Add navigation action tests**

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
  describe('navigation actions', () => {
    it('nextStep increments currentStep', () => {
      const store = useCharacterBuilderStore()
      store.nextStep()
      expect(store.currentStep).toBe(2)
    })

    it('nextStep does not exceed totalSteps', () => {
      const store = useCharacterBuilderStore()
      // Go to last step (6 for non-caster)
      for (let i = 0; i < 10; i++) {
        store.nextStep()
      }
      expect(store.currentStep).toBe(6)
    })

    it('previousStep decrements currentStep', () => {
      const store = useCharacterBuilderStore()
      store.currentStep = 3
      store.previousStep()
      expect(store.currentStep).toBe(2)
    })

    it('previousStep does not go below 1', () => {
      const store = useCharacterBuilderStore()
      store.previousStep()
      expect(store.currentStep).toBe(1)
    })

    it('goToStep navigates to valid step', () => {
      const store = useCharacterBuilderStore()
      store.goToStep(4)
      expect(store.currentStep).toBe(4)
    })

    it('goToStep ignores invalid step numbers', () => {
      const store = useCharacterBuilderStore()
      store.goToStep(0)
      expect(store.currentStep).toBe(1)
      store.goToStep(10)
      expect(store.currentStep).toBe(1)
    })
  })
```

**Step 2.6: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts`

Expected: All 13 tests PASS

**Step 2.7: Add reset action test**

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
  describe('reset action', () => {
    it('resets all state to initial values', () => {
      const store = useCharacterBuilderStore()

      // Modify state
      store.characterId = 123
      store.name = 'Gandalf'
      store.currentStep = 5
      store.raceId = 1
      store.classId = 2
      store.abilityScores.strength = 18

      // Reset
      store.reset()

      // Verify all reset
      expect(store.characterId).toBeNull()
      expect(store.name).toBe('')
      expect(store.currentStep).toBe(1)
      expect(store.raceId).toBeNull()
      expect(store.classId).toBeNull()
      expect(store.abilityScores.strength).toBe(10)
    })
  })
```

**Step 2.8: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts`

Expected: All 14 tests PASS

**Step 2.9: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat(character): add character builder Pinia store

- Wizard navigation state (currentStep, totalSteps)
- Character data fields (name, race, class, abilities, etc.)
- Navigation actions (nextStep, previousStep, goToStep)
- Reset action to clear wizard state
- 14 passing tests"
```

---

## Task 3: Create Wizard Container Page

**Files:**
- Create: `app/pages/characters/create.vue`
- Create: `tests/pages/characters/create.test.ts`

**Step 3.1: Write failing test for page rendering**

```typescript
// tests/pages/characters/create.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import CharacterCreatePage from '~/pages/characters/create.vue'

describe('CharacterCreatePage', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders the page title', async () => {
    const wrapper = await mountSuspended(CharacterCreatePage)
    expect(wrapper.text()).toContain('Create Your Character')
  })

  it('displays the stepper component', async () => {
    const wrapper = await mountSuspended(CharacterCreatePage)
    expect(wrapper.findComponent({ name: 'CharacterBuilderStepper' }).exists()).toBe(true)
  })

  it('shows step 1 content initially', async () => {
    const wrapper = await mountSuspended(CharacterCreatePage)
    expect(wrapper.findComponent({ name: 'CharacterBuilderStepName' }).exists()).toBe(true)
  })
})
```

**Step 3.2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/create.test.ts`

Expected: FAIL - Cannot find module

**Step 3.3: Create the page component**

```vue
<!-- app/pages/characters/create.vue -->
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

/**
 * Character Creation Wizard Page
 *
 * Multi-step wizard for creating D&D 5e characters.
 * Uses UStepper for navigation and Pinia store for state.
 */

// Page metadata
useSeoMeta({
  title: 'Create Your Character',
  description: 'Build your D&D 5e character step by step'
})

// Store
const store = useCharacterBuilderStore()
const { currentStep, totalSteps, isFirstStep, isLastStep, isCaster } = storeToRefs(store)

// Step definitions
const steps = computed(() => {
  const baseSteps = [
    { id: 1, name: 'name', label: 'Name', icon: 'i-heroicons-user' },
    { id: 2, name: 'race', label: 'Race', icon: 'i-heroicons-globe-alt' },
    { id: 3, name: 'class', label: 'Class', icon: 'i-heroicons-shield-check' },
    { id: 4, name: 'abilities', label: 'Abilities', icon: 'i-heroicons-chart-bar' },
    { id: 5, name: 'background', label: 'Background', icon: 'i-heroicons-book-open' }
  ]

  if (isCaster.value) {
    baseSteps.push({ id: 6, name: 'spells', label: 'Spells', icon: 'i-heroicons-sparkles' })
  }

  baseSteps.push({
    id: isCaster.value ? 7 : 6,
    name: 'review',
    label: 'Review',
    icon: 'i-heroicons-check-circle'
  })

  return baseSteps
})

// Reset wizard when leaving page
onBeforeUnmount(() => {
  // Only reset if character wasn't completed
  if (!store.isComplete) {
    // Optionally prompt user about unsaved progress
    // For now, just leave state (they might come back)
  }
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <!-- Page Header -->
    <div class="text-center mb-8">
      <h1 class="text-3xl font-bold text-gray-900 dark:text-white">
        Create Your Character
      </h1>
      <p class="mt-2 text-gray-600 dark:text-gray-400">
        Build your hero step by step
      </p>
    </div>

    <!-- Stepper Navigation -->
    <CharacterBuilderStepper
      :steps="steps"
      :current-step="currentStep"
      class="mb-8"
    />

    <!-- Step Content -->
    <div class="bg-white dark:bg-gray-800 rounded-lg shadow-sm p-6">
      <!-- Step 1: Name -->
      <CharacterBuilderStepName v-if="currentStep === 1" />

      <!-- Step 2: Race -->
      <CharacterBuilderStepRace v-else-if="currentStep === 2" />

      <!-- Step 3: Class -->
      <CharacterBuilderStepClass v-else-if="currentStep === 3" />

      <!-- Step 4: Abilities -->
      <CharacterBuilderStepAbilities v-else-if="currentStep === 4" />

      <!-- Step 5: Background -->
      <CharacterBuilderStepBackground v-else-if="currentStep === 5" />

      <!-- Step 6: Spells (conditional) or Review -->
      <template v-else-if="currentStep === 6">
        <CharacterBuilderStepSpells v-if="isCaster" />
        <CharacterBuilderStepReview v-else />
      </template>

      <!-- Step 7: Review (for casters) -->
      <CharacterBuilderStepReview v-else-if="currentStep === 7" />
    </div>

    <!-- Navigation Buttons -->
    <div class="flex justify-between mt-6">
      <UButton
        v-if="!isFirstStep"
        variant="outline"
        icon="i-heroicons-arrow-left"
        @click="store.previousStep()"
      >
        Back
      </UButton>
      <div v-else />

      <UButton
        v-if="!isLastStep"
        icon="i-heroicons-arrow-right"
        trailing
        @click="store.nextStep()"
      >
        Next
      </UButton>
    </div>
  </div>
</template>
```

**Step 3.4: Create placeholder step components**

Create stub components so the page renders:

```vue
<!-- app/components/character/builder/Stepper.vue -->
<script setup lang="ts">
interface Step {
  id: number
  name: string
  label: string
  icon: string
}

defineProps<{
  steps: Step[]
  currentStep: number
}>()
</script>

<template>
  <div class="flex justify-between">
    <div
      v-for="step in steps"
      :key="step.id"
      class="flex items-center"
      :class="{ 'text-primary': step.id === currentStep }"
    >
      <UIcon :name="step.icon" class="w-5 h-5 mr-1" />
      <span class="text-sm">{{ step.label }}</span>
    </div>
  </div>
</template>
```

```vue
<!-- app/components/character/builder/StepName.vue -->
<script setup lang="ts">
// Placeholder - will be implemented in Task 4
</script>

<template>
  <div>
    <h2 class="text-xl font-semibold mb-4">Name Your Character</h2>
    <p class="text-gray-500">Step 1 content coming soon...</p>
  </div>
</template>
```

Create similar placeholder files for:
- `app/components/character/builder/StepRace.vue`
- `app/components/character/builder/StepClass.vue`
- `app/components/character/builder/StepAbilities.vue`
- `app/components/character/builder/StepBackground.vue`
- `app/components/character/builder/StepSpells.vue`
- `app/components/character/builder/StepReview.vue`

Each with template:
```vue
<script setup lang="ts">
// Placeholder
</script>

<template>
  <div>
    <h2 class="text-xl font-semibold mb-4">[Step Name]</h2>
    <p class="text-gray-500">Coming soon...</p>
  </div>
</template>
```

**Step 3.5: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/create.test.ts`

Expected: All 3 tests PASS

**Step 3.6: Commit**

```bash
git add app/pages/characters/create.vue app/components/character/builder/
git commit -m "feat(character): add wizard container page and step placeholders

- Create page at /characters/create
- Stepper component for navigation
- Placeholder components for all 7 steps
- Navigation buttons with store integration
- 3 passing tests"
```

---

## Task 4: Implement Step 1 - Name Input

**Files:**
- Modify: `app/components/character/builder/StepName.vue`
- Create: `tests/components/character/builder/StepName.test.ts`

**Step 4.1: Write failing tests for StepName**

```typescript
// tests/components/character/builder/StepName.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepName from '~/components/character/builder/StepName.vue'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

// Mock apiFetch
vi.mock('~/composables/useApiFetch', () => ({
  apiFetch: vi.fn()
}))

describe('CharacterBuilderStepName', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.clearAllMocks()
  })

  it('renders the name input', async () => {
    const wrapper = await mountSuspended(StepName)
    const input = wrapper.find('input[type="text"]')
    expect(input.exists()).toBe(true)
  })

  it('shows placeholder text', async () => {
    const wrapper = await mountSuspended(StepName)
    const input = wrapper.find('input')
    expect(input.attributes('placeholder')).toContain('name')
  })

  it('binds to store name value', async () => {
    const store = useCharacterBuilderStore()
    store.name = 'Gandalf'

    const wrapper = await mountSuspended(StepName)
    const input = wrapper.find('input')

    expect((input.element as HTMLInputElement).value).toBe('Gandalf')
  })

  it('disables create button when name is empty', async () => {
    const wrapper = await mountSuspended(StepName)
    const button = wrapper.find('button')

    expect(button.attributes('disabled')).toBeDefined()
  })

  it('enables create button when name is provided', async () => {
    const store = useCharacterBuilderStore()
    store.name = 'Frodo'

    const wrapper = await mountSuspended(StepName)
    await wrapper.vm.$nextTick()

    const button = wrapper.find('button')
    expect(button.attributes('disabled')).toBeUndefined()
  })
})
```

**Step 4.2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepName.test.ts`

Expected: FAIL - Tests fail because component is placeholder

**Step 4.3: Implement StepName component**

```vue
<!-- app/components/character/builder/StepName.vue -->
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

/**
 * Step 1: Name Your Character
 *
 * Simple name input with validation.
 * Creates a draft character on the API when user proceeds.
 */

const store = useCharacterBuilderStore()
const { name, isLoading, characterId } = storeToRefs(store)

// Validation
const isValid = computed(() => name.value.trim().length > 0)
const errorMessage = ref<string | null>(null)

// Create character and proceed
async function handleCreate() {
  if (!isValid.value) return

  errorMessage.value = null
  store.isLoading = true

  try {
    const response = await apiFetch<{ data: { id: number; name: string } }>('/characters', {
      method: 'POST',
      body: { name: name.value.trim() }
    })

    store.characterId = response.data.id
    store.nextStep()
  }
  catch (err: unknown) {
    errorMessage.value = err instanceof Error ? err.message : 'Failed to create character'
  }
  finally {
    store.isLoading = false
  }
}

// If character already exists, just proceed
function handleNext() {
  if (characterId.value) {
    store.nextStep()
  }
  else {
    handleCreate()
  }
}
</script>

<template>
  <div class="max-w-md mx-auto space-y-6">
    <!-- Header -->
    <div class="text-center">
      <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
        Name Your Character
      </h2>
      <p class="mt-2 text-gray-600 dark:text-gray-400">
        What shall this hero be called?
      </p>
    </div>

    <!-- Name Input -->
    <UFormField
      label="Character Name"
      :error="errorMessage"
      required
    >
      <UInput
        v-model="name"
        type="text"
        placeholder="Enter a name..."
        size="xl"
        autofocus
        :disabled="isLoading"
        @keyup.enter="handleNext"
      />
    </UFormField>

    <!-- Helper Text -->
    <p class="text-sm text-gray-500 dark:text-gray-400 text-center">
      Choose a name that fits your character's personality and background.
      You can always change it later.
    </p>

    <!-- Create Button -->
    <div class="flex justify-center">
      <UButton
        size="lg"
        :disabled="!isValid"
        :loading="isLoading"
        @click="handleNext"
      >
        <template v-if="characterId">
          Continue
        </template>
        <template v-else>
          Begin Your Journey
        </template>
      </UButton>
    </div>
  </div>
</template>
```

**Step 4.4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepName.test.ts`

Expected: All 5 tests PASS

**Step 4.5: Commit**

```bash
git add app/components/character/builder/StepName.vue tests/components/character/builder/StepName.test.ts
git commit -m "feat(character): implement Step 1 - Name input

- Text input bound to store name
- Validation (non-empty)
- Creates draft character via API
- Loading state during API call
- 5 passing tests"
```

---

## Task 5: Add createDraft Store Action with API Integration

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Modify: `tests/stores/characterBuilder.test.ts`

**Step 5.1: Add tests for createDraft action**

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
import { vi } from 'vitest'

// Mock apiFetch at the top of the file
const mockApiFetch = vi.fn()
vi.mock('~/composables/useApiFetch', () => ({
  apiFetch: (...args: unknown[]) => mockApiFetch(...args)
}))

// Add this describe block:
describe('createDraft action', () => {
  beforeEach(() => {
    mockApiFetch.mockReset()
  })

  it('calls API with character name', async () => {
    mockApiFetch.mockResolvedValue({ data: { id: 42, name: 'Gandalf' } })

    const store = useCharacterBuilderStore()
    await store.createDraft('Gandalf')

    expect(mockApiFetch).toHaveBeenCalledWith('/characters', {
      method: 'POST',
      body: { name: 'Gandalf' }
    })
  })

  it('sets characterId from response', async () => {
    mockApiFetch.mockResolvedValue({ data: { id: 42, name: 'Gandalf' } })

    const store = useCharacterBuilderStore()
    await store.createDraft('Gandalf')

    expect(store.characterId).toBe(42)
  })

  it('sets name from input', async () => {
    mockApiFetch.mockResolvedValue({ data: { id: 42, name: 'Gandalf' } })

    const store = useCharacterBuilderStore()
    await store.createDraft('Gandalf')

    expect(store.name).toBe('Gandalf')
  })

  it('sets loading state during API call', async () => {
    let resolvePromise: (value: unknown) => void
    mockApiFetch.mockReturnValue(new Promise(resolve => {
      resolvePromise = resolve
    }))

    const store = useCharacterBuilderStore()
    const promise = store.createDraft('Gandalf')

    expect(store.isLoading).toBe(true)

    resolvePromise!({ data: { id: 42, name: 'Gandalf' } })
    await promise

    expect(store.isLoading).toBe(false)
  })

  it('sets error on API failure', async () => {
    mockApiFetch.mockRejectedValue(new Error('Network error'))

    const store = useCharacterBuilderStore()

    await expect(store.createDraft('Gandalf')).rejects.toThrow('Network error')
    expect(store.error).toBe('Network error')
  })
})
```

**Step 5.2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts`

Expected: FAIL - createDraft is not a function

**Step 5.3: Add createDraft action to store**

Add to `app/stores/characterBuilder.ts`:

```typescript
  // ══════════════════════════════════════════════════════════════
  // API ACTIONS
  // ══════════════════════════════════════════════════════════════

  /**
   * Step 1: Create a draft character with just a name
   */
  async function createDraft(characterName: string): Promise<void> {
    isLoading.value = true
    error.value = null

    try {
      const response = await apiFetch<{ data: { id: number; name: string } }>('/characters', {
        method: 'POST',
        body: { name: characterName }
      })

      characterId.value = response.data.id
      name.value = characterName
    }
    catch (err: unknown) {
      error.value = err instanceof Error ? err.message : 'Failed to create character'
      throw err
    }
    finally {
      isLoading.value = false
    }
  }
```

And add to the return statement:
```typescript
    createDraft,
```

**Step 5.4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts`

Expected: All 19 tests PASS

**Step 5.5: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat(character): add createDraft store action

- POST /characters with name
- Sets characterId and name from response
- Loading state management
- Error handling
- 5 new tests (19 total)"
```

---

## Task 6: Create Character List Page

**Files:**
- Create: `app/pages/characters/index.vue`
- Create: `app/components/character/Card.vue`
- Create: `tests/pages/characters/index.test.ts`

**Step 6.1: Write failing test for list page**

```typescript
// tests/pages/characters/index.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import CharacterListPage from '~/pages/characters/index.vue'

// Mock useAsyncData
vi.mock('#app', async () => {
  const actual = await vi.importActual('#app')
  return {
    ...actual,
    useAsyncData: vi.fn().mockReturnValue({
      data: ref({ data: [] }),
      pending: ref(false),
      error: ref(null),
      refresh: vi.fn()
    })
  }
})

describe('CharacterListPage', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders the page title', async () => {
    const wrapper = await mountSuspended(CharacterListPage)
    expect(wrapper.text()).toContain('Your Characters')
  })

  it('shows create button', async () => {
    const wrapper = await mountSuspended(CharacterListPage)
    expect(wrapper.text()).toContain('Create Character')
  })

  it('shows empty state when no characters', async () => {
    const wrapper = await mountSuspended(CharacterListPage)
    expect(wrapper.text()).toContain('No characters yet')
  })
})
```

**Step 6.2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/index.test.ts`

Expected: FAIL - Cannot find module

**Step 6.3: Create the list page**

```vue
<!-- app/pages/characters/index.vue -->
<script setup lang="ts">
import type { CharacterSummary } from '~/types'

/**
 * Character List Page
 *
 * Displays user's characters with option to create new ones.
 */

useSeoMeta({
  title: 'Your Characters',
  description: 'Manage your D&D 5e characters'
})

// Fetch characters
const { data, pending, error, refresh } = await useAsyncData(
  'characters-list',
  () => apiFetch<{ data: CharacterSummary[] }>('/characters')
)

const characters = computed(() => data.value?.data ?? [])

// Delete character
async function deleteCharacter(id: number) {
  if (!confirm('Are you sure you want to delete this character?')) return

  try {
    await apiFetch(`/characters/${id}`, { method: 'DELETE' })
    await refresh()
  }
  catch (err) {
    console.error('Failed to delete character:', err)
  }
}
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <!-- Header -->
    <div class="flex items-center justify-between mb-8">
      <div>
        <h1 class="text-3xl font-bold text-gray-900 dark:text-white">
          Your Characters
        </h1>
        <p class="mt-1 text-gray-600 dark:text-gray-400">
          Manage your D&D 5e heroes
        </p>
      </div>

      <UButton
        to="/characters/create"
        icon="i-heroicons-plus"
        size="lg"
      >
        Create Character
      </UButton>
    </div>

    <!-- Loading State -->
    <div
      v-if="pending"
      class="flex justify-center py-12"
    >
      <UIcon
        name="i-heroicons-arrow-path"
        class="w-8 h-8 animate-spin text-gray-400"
      />
    </div>

    <!-- Error State -->
    <UAlert
      v-else-if="error"
      color="red"
      icon="i-heroicons-exclamation-triangle"
      title="Failed to load characters"
      :description="error.message"
    />

    <!-- Empty State -->
    <div
      v-else-if="characters.length === 0"
      class="text-center py-12"
    >
      <UIcon
        name="i-heroicons-user-group"
        class="w-16 h-16 mx-auto text-gray-300 dark:text-gray-600"
      />
      <h3 class="mt-4 text-lg font-medium text-gray-900 dark:text-white">
        No characters yet
      </h3>
      <p class="mt-2 text-gray-500 dark:text-gray-400">
        Create your first character to begin your adventure!
      </p>
      <UButton
        to="/characters/create"
        class="mt-6"
        icon="i-heroicons-plus"
      >
        Create Your First Character
      </UButton>
    </div>

    <!-- Character Grid -->
    <div
      v-else
      class="grid gap-4 sm:grid-cols-2 lg:grid-cols-3"
    >
      <CharacterCard
        v-for="character in characters"
        :key="character.id"
        :character="character"
        @delete="deleteCharacter(character.id)"
      />
    </div>
  </div>
</template>
```

**Step 6.4: Create CharacterCard component**

```vue
<!-- app/components/character/Card.vue -->
<script setup lang="ts">
import type { CharacterSummary } from '~/types'

const props = defineProps<{
  character: CharacterSummary
}>()

defineEmits<{
  delete: []
}>()

const statusColor = computed(() =>
  props.character.is_complete ? 'green' : 'yellow'
)

const statusText = computed(() =>
  props.character.is_complete ? 'Complete' : 'Draft'
)
</script>

<template>
  <UCard class="hover:shadow-md transition-shadow">
    <template #header>
      <div class="flex items-center justify-between">
        <h3 class="font-semibold text-lg truncate">
          {{ character.name }}
        </h3>
        <UBadge
          :color="statusColor"
          variant="subtle"
          size="sm"
        >
          {{ statusText }}
        </UBadge>
      </div>
    </template>

    <div class="space-y-2 text-sm text-gray-600 dark:text-gray-400">
      <div class="flex items-center gap-2">
        <UIcon
          name="i-heroicons-globe-alt"
          class="w-4 h-4"
        />
        <span>{{ character.race?.name ?? 'No race selected' }}</span>
      </div>
      <div class="flex items-center gap-2">
        <UIcon
          name="i-heroicons-shield-check"
          class="w-4 h-4"
        />
        <span>{{ character.class?.name ?? 'No class selected' }}</span>
      </div>
      <div class="flex items-center gap-2">
        <UIcon
          name="i-heroicons-arrow-trending-up"
          class="w-4 h-4"
        />
        <span>Level {{ character.level }}</span>
      </div>
    </div>

    <template #footer>
      <div class="flex justify-between">
        <UButton
          :to="`/characters/${character.id}`"
          variant="ghost"
          size="sm"
        >
          View
        </UButton>
        <UButton
          variant="ghost"
          color="red"
          size="sm"
          icon="i-heroicons-trash"
          @click="$emit('delete')"
        />
      </div>
    </template>
  </UCard>
</template>
```

**Step 6.5: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/index.test.ts`

Expected: All 3 tests PASS

**Step 6.6: Commit**

```bash
git add app/pages/characters/index.vue app/components/character/Card.vue tests/pages/characters/index.test.ts
git commit -m "feat(character): add character list page

- Lists all characters from API
- Empty state for new users
- CharacterCard component with status badge
- Delete functionality
- Navigation to create page
- 3 passing tests"
```

---

## Task 7: Run Full Test Suite and Fix Any Issues

**Step 7.1: Run the core test suite**

Run: `docker compose exec nuxt npm run test:core`

Expected: All tests pass

**Step 7.2: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No errors

**Step 7.3: Run linting**

Run: `docker compose exec nuxt npm run lint:fix`

Expected: No errors or auto-fixed

**Step 7.4: Final commit for Phase 1**

```bash
git add -A
git commit -m "chore: Phase 1 complete - lint and type fixes

All tests passing, types valid, lint clean"
```

---

## Phase 1 Summary

**Files Created:**
- `app/types/character.ts` - TypeScript types
- `app/stores/characterBuilder.ts` - Pinia store
- `app/pages/characters/index.vue` - List page
- `app/pages/characters/create.vue` - Wizard page
- `app/components/character/Card.vue` - List card
- `app/components/character/builder/Stepper.vue` - Navigation
- `app/components/character/builder/StepName.vue` - Step 1
- `app/components/character/builder/Step*.vue` - Placeholders for steps 2-7

**Tests Created:**
- `tests/stores/characterBuilder.test.ts` - 19 tests
- `tests/components/character/builder/StepName.test.ts` - 5 tests
- `tests/pages/characters/create.test.ts` - 3 tests
- `tests/pages/characters/index.test.ts` - 3 tests

**Total: ~30 tests**

**Next Phase:** Step 2 (Race Selection) and Step 3 (Class Selection)
