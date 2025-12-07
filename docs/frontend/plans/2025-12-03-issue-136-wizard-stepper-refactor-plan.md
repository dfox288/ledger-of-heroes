# Wizard Stepper Refactor Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Convert the character builder from a single page with numbered steps to route-based steps with a compact progress bar UI.

**Architecture:** Each wizard step becomes a nested route under `/characters/[id]/edit/[step]`. A step registry defines all steps with visibility conditions. Navigation composable handles prev/next based on active steps. Route middleware guards conditional steps.

**Tech Stack:** Nuxt 4 file-based routing, Vue 3 Composition API, Pinia, TypeScript

**Design Doc:** `docs/plans/2025-12-03-issue-136-wizard-stepper-refactor-design.md`

---

## Task 1: Create Step Registry Type and Composable

**Files:**
- Create: `app/composables/useWizardSteps.ts`
- Test: `tests/composables/useWizardSteps.test.ts`

**Step 1: Write the failing test**

Create `tests/composables/useWizardSteps.test.ts`:

```typescript
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'

// Mock the store before importing the composable
vi.mock('~/stores/characterBuilder', () => ({
  useCharacterBuilderStore: vi.fn(() => ({
    hasPendingChoices: false,
    isCaster: false
  }))
}))

describe('useWizardSteps', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    vi.clearAllMocks()
  })

  describe('stepRegistry', () => {
    it('exports a step registry array', async () => {
      const { stepRegistry } = await import('~/composables/useWizardSteps')
      expect(Array.isArray(stepRegistry)).toBe(true)
      expect(stepRegistry.length).toBeGreaterThan(0)
    })

    it('each step has required properties', async () => {
      const { stepRegistry } = await import('~/composables/useWizardSteps')
      for (const step of stepRegistry) {
        expect(step).toHaveProperty('name')
        expect(step).toHaveProperty('label')
        expect(step).toHaveProperty('icon')
        expect(step).toHaveProperty('visible')
        expect(typeof step.visible).toBe('function')
      }
    })

    it('includes core steps: name, race, class, abilities, background, equipment, review', async () => {
      const { stepRegistry } = await import('~/composables/useWizardSteps')
      const stepNames = stepRegistry.map(s => s.name)
      expect(stepNames).toContain('name')
      expect(stepNames).toContain('race')
      expect(stepNames).toContain('class')
      expect(stepNames).toContain('abilities')
      expect(stepNames).toContain('background')
      expect(stepNames).toContain('equipment')
      expect(stepNames).toContain('review')
    })

    it('includes conditional steps: proficiencies, spells', async () => {
      const { stepRegistry } = await import('~/composables/useWizardSteps')
      const stepNames = stepRegistry.map(s => s.name)
      expect(stepNames).toContain('proficiencies')
      expect(stepNames).toContain('spells')
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useWizardSteps.test.ts`

Expected: FAIL with "Cannot find module '~/composables/useWizardSteps'"

**Step 3: Write minimal implementation**

Create `app/composables/useWizardSteps.ts`:

```typescript
// app/composables/useWizardSteps.ts
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

export interface WizardStep {
  name: string
  label: string
  icon: string
  visible: () => boolean
}

// Step registry - single source of truth for wizard steps
// Order matters: steps appear in this order in the wizard
export const stepRegistry: WizardStep[] = [
  {
    name: 'name',
    label: 'Name',
    icon: 'i-heroicons-user',
    visible: () => true
  },
  {
    name: 'race',
    label: 'Race',
    icon: 'i-heroicons-globe-alt',
    visible: () => true
  },
  {
    name: 'class',
    label: 'Class',
    icon: 'i-heroicons-shield-check',
    visible: () => true
  },
  {
    name: 'abilities',
    label: 'Abilities',
    icon: 'i-heroicons-chart-bar',
    visible: () => true
  },
  {
    name: 'background',
    label: 'Background',
    icon: 'i-heroicons-book-open',
    visible: () => true
  },
  {
    name: 'proficiencies',
    label: 'Proficiencies',
    icon: 'i-heroicons-academic-cap',
    visible: () => {
      const store = useCharacterBuilderStore()
      return store.hasPendingChoices
    }
  },
  {
    name: 'equipment',
    label: 'Equipment',
    icon: 'i-heroicons-briefcase',
    visible: () => true
  },
  {
    name: 'spells',
    label: 'Spells',
    icon: 'i-heroicons-sparkles',
    visible: () => {
      const store = useCharacterBuilderStore()
      return store.isCaster
    }
  },
  {
    name: 'review',
    label: 'Review',
    icon: 'i-heroicons-check-circle',
    visible: () => true
  }
]
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useWizardSteps.test.ts`

Expected: PASS

**Step 5: Commit**

```bash
git add app/composables/useWizardSteps.ts tests/composables/useWizardSteps.test.ts
git commit -m "feat(wizard): add step registry with visibility conditions (#136)"
```

---

## Task 2: Add Conditional Step Visibility Tests

**Files:**
- Modify: `tests/composables/useWizardSteps.test.ts`

**Step 1: Add tests for conditional visibility**

Add to `tests/composables/useWizardSteps.test.ts`:

```typescript
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

describe('conditional step visibility', () => {
  it('proficiencies step is hidden when no pending choices', async () => {
    vi.mocked(useCharacterBuilderStore).mockReturnValue({
      hasPendingChoices: false,
      isCaster: false
    } as any)

    const { stepRegistry } = await import('~/composables/useWizardSteps')
    const profStep = stepRegistry.find(s => s.name === 'proficiencies')
    expect(profStep?.visible()).toBe(false)
  })

  it('proficiencies step is visible when has pending choices', async () => {
    vi.mocked(useCharacterBuilderStore).mockReturnValue({
      hasPendingChoices: true,
      isCaster: false
    } as any)

    // Need to reimport to get fresh module
    vi.resetModules()
    const { stepRegistry } = await import('~/composables/useWizardSteps')
    const profStep = stepRegistry.find(s => s.name === 'proficiencies')
    expect(profStep?.visible()).toBe(true)
  })

  it('spells step is hidden for non-casters', async () => {
    vi.mocked(useCharacterBuilderStore).mockReturnValue({
      hasPendingChoices: false,
      isCaster: false
    } as any)

    vi.resetModules()
    const { stepRegistry } = await import('~/composables/useWizardSteps')
    const spellsStep = stepRegistry.find(s => s.name === 'spells')
    expect(spellsStep?.visible()).toBe(false)
  })

  it('spells step is visible for casters', async () => {
    vi.mocked(useCharacterBuilderStore).mockReturnValue({
      hasPendingChoices: false,
      isCaster: true
    } as any)

    vi.resetModules()
    const { stepRegistry } = await import('~/composables/useWizardSteps')
    const spellsStep = stepRegistry.find(s => s.name === 'spells')
    expect(spellsStep?.visible()).toBe(true)
  })

  it('core steps are always visible', async () => {
    vi.mocked(useCharacterBuilderStore).mockReturnValue({
      hasPendingChoices: false,
      isCaster: false
    } as any)

    vi.resetModules()
    const { stepRegistry } = await import('~/composables/useWizardSteps')
    const coreSteps = ['name', 'race', 'class', 'abilities', 'background', 'equipment', 'review']

    for (const stepName of coreSteps) {
      const step = stepRegistry.find(s => s.name === stepName)
      expect(step?.visible(), `${stepName} should be visible`).toBe(true)
    }
  })
})
```

**Step 2: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/composables/useWizardSteps.test.ts`

Expected: PASS (all tests)

**Step 3: Commit**

```bash
git add tests/composables/useWizardSteps.test.ts
git commit -m "test(wizard): add conditional step visibility tests (#136)"
```

---

## Task 3: Create Navigation Composable

**Files:**
- Modify: `app/composables/useWizardSteps.ts`
- Modify: `tests/composables/useWizardSteps.test.ts`

**Step 1: Write failing tests for navigation**

Add to `tests/composables/useWizardSteps.test.ts`:

```typescript
// Mock useRoute
const mockRoute = ref({ params: { id: '5', step: 'race' } })
vi.mock('#app', () => ({
  useRoute: () => mockRoute.value,
  navigateTo: vi.fn()
}))

describe('useWizardNavigation', () => {
  beforeEach(() => {
    vi.clearAllMocks()
    mockRoute.value = { params: { id: '5', step: 'race' } }

    // Default: non-caster, no pending choices
    vi.mocked(useCharacterBuilderStore).mockReturnValue({
      hasPendingChoices: false,
      isCaster: false,
      characterId: 5
    } as any)
  })

  it('returns active steps filtered by visibility', async () => {
    vi.resetModules()
    const { useWizardNavigation } = await import('~/composables/useWizardSteps')
    const { activeSteps } = useWizardNavigation()

    // With non-caster and no pending choices, should have 7 steps
    expect(activeSteps.value.length).toBe(7)
    expect(activeSteps.value.map(s => s.name)).not.toContain('proficiencies')
    expect(activeSteps.value.map(s => s.name)).not.toContain('spells')
  })

  it('computes current step index from route', async () => {
    mockRoute.value = { params: { id: '5', step: 'class' } }

    vi.resetModules()
    const { useWizardNavigation } = await import('~/composables/useWizardSteps')
    const { currentStepIndex } = useWizardNavigation()

    expect(currentStepIndex.value).toBe(2) // 0=name, 1=race, 2=class
  })

  it('computes totalSteps from active steps', async () => {
    vi.resetModules()
    const { useWizardNavigation } = await import('~/composables/useWizardSteps')
    const { totalSteps } = useWizardNavigation()

    expect(totalSteps.value).toBe(7)
  })

  it('isFirstStep is true on first step', async () => {
    mockRoute.value = { params: { id: '5', step: 'name' } }

    vi.resetModules()
    const { useWizardNavigation } = await import('~/composables/useWizardSteps')
    const { isFirstStep } = useWizardNavigation()

    expect(isFirstStep.value).toBe(true)
  })

  it('isLastStep is true on review step', async () => {
    mockRoute.value = { params: { id: '5', step: 'review' } }

    vi.resetModules()
    const { useWizardNavigation } = await import('~/composables/useWizardSteps')
    const { isLastStep } = useWizardNavigation()

    expect(isLastStep.value).toBe(true)
  })
})
```

**Step 2: Run tests to verify they fail**

Run: `docker compose exec nuxt npm run test -- tests/composables/useWizardSteps.test.ts`

Expected: FAIL with "useWizardNavigation is not a function"

**Step 3: Implement useWizardNavigation**

Add to `app/composables/useWizardSteps.ts`:

```typescript
/**
 * Composable for wizard navigation
 * Uses route params to track current step instead of store state
 */
export function useWizardNavigation() {
  const route = useRoute()
  const store = useCharacterBuilderStore()

  // Filter registry to only visible steps
  const activeSteps = computed(() =>
    stepRegistry.filter(step => step.visible())
  )

  // Current step from route
  const currentStepName = computed(() =>
    route.params.step as string || 'name'
  )

  const currentStepIndex = computed(() =>
    activeSteps.value.findIndex(s => s.name === currentStepName.value)
  )

  const currentStep = computed(() =>
    activeSteps.value[currentStepIndex.value]
  )

  const totalSteps = computed(() => activeSteps.value.length)
  const isFirstStep = computed(() => currentStepIndex.value === 0)
  const isLastStep = computed(() => currentStepIndex.value === totalSteps.value - 1)

  // Navigation functions
  async function nextStep() {
    const nextIndex = currentStepIndex.value + 1
    if (nextIndex < activeSteps.value.length) {
      const next = activeSteps.value[nextIndex]
      await navigateTo(`/characters/${store.characterId}/edit/${next.name}`)
    }
  }

  async function previousStep() {
    const prevIndex = currentStepIndex.value - 1
    if (prevIndex >= 0) {
      const prev = activeSteps.value[prevIndex]
      await navigateTo(`/characters/${store.characterId}/edit/${prev.name}`)
    }
  }

  async function goToStep(stepName: string) {
    const step = activeSteps.value.find(s => s.name === stepName)
    if (step) {
      await navigateTo(`/characters/${store.characterId}/edit/${stepName}`)
    }
  }

  return {
    stepRegistry,
    activeSteps,
    currentStep,
    currentStepName,
    currentStepIndex,
    totalSteps,
    isFirstStep,
    isLastStep,
    nextStep,
    previousStep,
    goToStep
  }
}
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/composables/useWizardSteps.test.ts`

Expected: PASS

**Step 5: Commit**

```bash
git add app/composables/useWizardSteps.ts tests/composables/useWizardSteps.test.ts
git commit -m "feat(wizard): add navigation composable with route-based tracking (#136)"
```

---

## Task 4: Create Compact Progress Bar Component

**Files:**
- Create: `app/components/character/builder/ProgressBar.vue`
- Create: `tests/components/character/builder/ProgressBar.test.ts`

**Step 1: Write failing test**

Create `tests/components/character/builder/ProgressBar.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import ProgressBar from '~/components/character/builder/ProgressBar.vue'

const mockSteps = [
  { name: 'name', label: 'Name', icon: 'i-heroicons-user', visible: () => true },
  { name: 'race', label: 'Race', icon: 'i-heroicons-globe-alt', visible: () => true },
  { name: 'class', label: 'Class', icon: 'i-heroicons-shield-check', visible: () => true },
  { name: 'review', label: 'Review', icon: 'i-heroicons-check-circle', visible: () => true }
]

describe('ProgressBar', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('displays step counter with current step', async () => {
    const wrapper = await mountSuspended(ProgressBar, {
      props: {
        steps: mockSteps,
        currentIndex: 1, // race
        currentLabel: 'Race'
      }
    })

    expect(wrapper.text()).toContain('Step 2 of 4')
    expect(wrapper.text()).toContain('Race')
  })

  it('renders progress dots for each step', async () => {
    const wrapper = await mountSuspended(ProgressBar, {
      props: {
        steps: mockSteps,
        currentIndex: 1,
        currentLabel: 'Race'
      }
    })

    const dots = wrapper.findAll('[data-test="progress-dot"]')
    expect(dots.length).toBe(4)
  })

  it('marks completed steps with filled style', async () => {
    const wrapper = await mountSuspended(ProgressBar, {
      props: {
        steps: mockSteps,
        currentIndex: 2, // class (0=name, 1=race completed)
        currentLabel: 'Class'
      }
    })

    const dots = wrapper.findAll('[data-test="progress-dot"]')
    // First two dots should be completed
    expect(dots[0].classes()).toContain('bg-primary')
    expect(dots[1].classes()).toContain('bg-primary')
  })

  it('emits step-click when clicking a completed dot', async () => {
    const wrapper = await mountSuspended(ProgressBar, {
      props: {
        steps: mockSteps,
        currentIndex: 2,
        currentLabel: 'Class'
      }
    })

    const dots = wrapper.findAll('[data-test="progress-dot"]')
    await dots[0].trigger('click') // Click first (completed) dot

    expect(wrapper.emitted('step-click')).toBeTruthy()
    expect(wrapper.emitted('step-click')![0]).toEqual(['name'])
  })

  it('does not emit step-click when clicking future dot', async () => {
    const wrapper = await mountSuspended(ProgressBar, {
      props: {
        steps: mockSteps,
        currentIndex: 1,
        currentLabel: 'Race'
      }
    })

    const dots = wrapper.findAll('[data-test="progress-dot"]')
    await dots[3].trigger('click') // Click last (future) dot

    expect(wrapper.emitted('step-click')).toBeFalsy()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/ProgressBar.test.ts`

Expected: FAIL with "Cannot find module"

**Step 3: Create ProgressBar component**

Create `app/components/character/builder/ProgressBar.vue`:

```vue
<!-- app/components/character/builder/ProgressBar.vue -->
<script setup lang="ts">
import type { WizardStep } from '~/composables/useWizardSteps'

const props = defineProps<{
  steps: WizardStep[]
  currentIndex: number
  currentLabel: string
}>()

const emit = defineEmits<{
  'step-click': [stepName: string]
}>()

function handleDotClick(step: WizardStep, index: number) {
  // Only allow clicking completed steps (before current)
  if (index < props.currentIndex) {
    emit('step-click', step.name)
  }
}

function getDotClasses(index: number): string[] {
  const classes = ['w-3', 'h-3', 'rounded-full', 'transition-all']

  if (index < props.currentIndex) {
    // Completed
    classes.push('bg-primary', 'cursor-pointer', 'hover:scale-110')
  } else if (index === props.currentIndex) {
    // Current
    classes.push('bg-primary', 'ring-2', 'ring-primary', 'ring-offset-2', 'ring-offset-white', 'dark:ring-offset-gray-900')
  } else {
    // Future
    classes.push('bg-gray-300', 'dark:bg-gray-600', 'cursor-not-allowed')
  }

  return classes
}

// Human-readable step number (1-indexed)
const stepNumber = computed(() => props.currentIndex + 1)
const totalSteps = computed(() => props.steps.length)
</script>

<template>
  <div class="flex items-center justify-between gap-4 py-3 px-4 bg-gray-50 dark:bg-gray-800/50 rounded-lg">
    <!-- Progress dots -->
    <div class="flex items-center gap-2">
      <button
        v-for="(step, index) in steps"
        :key="step.name"
        type="button"
        data-test="progress-dot"
        :class="getDotClasses(index)"
        :disabled="index >= currentIndex"
        :title="step.label"
        @click="handleDotClick(step, index)"
      />
    </div>

    <!-- Step counter -->
    <div class="text-sm text-gray-600 dark:text-gray-400">
      <span class="font-medium text-gray-900 dark:text-white">
        Step {{ stepNumber }} of {{ totalSteps }}:
      </span>
      {{ currentLabel }}
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/ProgressBar.test.ts`

Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/ProgressBar.vue tests/components/character/builder/ProgressBar.test.ts
git commit -m "feat(wizard): add compact progress bar component (#136)"
```

---

## Task 5: Create Route Middleware for Step Guards

**Files:**
- Create: `app/middleware/wizard-step.ts`
- Create: `tests/middleware/wizard-step.test.ts`

**Step 1: Write failing test**

Create `tests/middleware/wizard-step.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'

// Mock navigateTo
const mockNavigateTo = vi.fn()
vi.mock('#app', () => ({
  navigateTo: mockNavigateTo,
  defineNuxtRouteMiddleware: (fn: any) => fn
}))

// Mock the store
vi.mock('~/stores/characterBuilder', () => ({
  useCharacterBuilderStore: vi.fn(() => ({
    hasPendingChoices: false,
    isCaster: false,
    characterId: 5
  }))
}))

describe('wizard-step middleware', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    mockNavigateTo.mockClear()
  })

  it('allows navigation to always-visible steps', async () => {
    const { default: middleware } = await import('~/middleware/wizard-step')

    const to = { params: { id: '5', step: 'name' } } as any
    const result = await middleware(to, {} as any)

    expect(result).toBeUndefined() // No redirect
    expect(mockNavigateTo).not.toHaveBeenCalled()
  })

  it('redirects from spells step when not a caster', async () => {
    const { useCharacterBuilderStore } = await import('~/stores/characterBuilder')
    vi.mocked(useCharacterBuilderStore).mockReturnValue({
      hasPendingChoices: false,
      isCaster: false,
      characterId: 5
    } as any)

    vi.resetModules()
    const { default: middleware } = await import('~/middleware/wizard-step')

    const to = { params: { id: '5', step: 'spells' } } as any
    await middleware(to, {} as any)

    expect(mockNavigateTo).toHaveBeenCalledWith('/characters/5/edit/name')
  })

  it('allows navigation to spells step when caster', async () => {
    const { useCharacterBuilderStore } = await import('~/stores/characterBuilder')
    vi.mocked(useCharacterBuilderStore).mockReturnValue({
      hasPendingChoices: false,
      isCaster: true,
      characterId: 5
    } as any)

    vi.resetModules()
    const { default: middleware } = await import('~/middleware/wizard-step')

    const to = { params: { id: '5', step: 'spells' } } as any
    const result = await middleware(to, {} as any)

    expect(result).toBeUndefined()
  })

  it('redirects from unknown step', async () => {
    vi.resetModules()
    const { default: middleware } = await import('~/middleware/wizard-step')

    const to = { params: { id: '5', step: 'unknown-step' } } as any
    await middleware(to, {} as any)

    expect(mockNavigateTo).toHaveBeenCalledWith('/characters/5/edit/name')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/middleware/wizard-step.test.ts`

Expected: FAIL

**Step 3: Create middleware**

Create `app/middleware/wizard-step.ts`:

```typescript
// app/middleware/wizard-step.ts
import { stepRegistry } from '~/composables/useWizardSteps'

export default defineNuxtRouteMiddleware((to) => {
  const stepName = to.params.step as string

  // Find the step in registry
  const step = stepRegistry.find(s => s.name === stepName)

  // If step doesn't exist or isn't currently visible, redirect to first step
  if (!step || !step.visible()) {
    const characterId = to.params.id
    return navigateTo(`/characters/${characterId}/edit/name`)
  }
})
```

**Step 4: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/middleware/wizard-step.test.ts`

Expected: PASS

**Step 5: Commit**

```bash
git add app/middleware/wizard-step.ts tests/middleware/wizard-step.test.ts
git commit -m "feat(wizard): add route middleware for step guards (#136)"
```

---

## Task 6: Create Edit Layout Page

**Files:**
- Modify: `app/pages/characters/[id]/edit.vue`

**Step 1: Refactor edit.vue to be a layout wrapper**

Replace contents of `app/pages/characters/[id]/edit.vue`:

```vue
<!-- app/pages/characters/[id]/edit.vue -->
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'
import { useWizardNavigation } from '~/composables/useWizardSteps'

definePageMeta({
  middleware: ['wizard-step']
})

const route = useRoute()
const characterId = computed(() => Number(route.params.id))
const isNewCharacter = computed(() => route.query.new === 'true')

// Page metadata
useSeoMeta({
  title: 'Edit Character',
  description: 'Continue building your D&D 5e character'
})

// Store
const store = useCharacterBuilderStore()
const { isLoading, error, name } = storeToRefs(store)

// Wizard navigation
const {
  activeSteps,
  currentStep,
  currentStepIndex,
  isFirstStep,
  isLastStep,
  nextStep,
  previousStep,
  goToStep
} = useWizardNavigation()

// Load character on mount
onMounted(async () => {
  store.reset()

  try {
    await store.loadCharacterForEditing(characterId.value)

    // For new characters, redirect to name step
    if (isNewCharacter.value && route.params.step !== 'name') {
      await navigateTo(`/characters/${characterId.value}/edit/name`)
    }
  } catch {
    await navigateTo(`/characters/${characterId.value}`)
  }
})

// Handle progress bar step click
function handleStepClick(stepName: string) {
  goToStep(stepName)
}
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <!-- Loading State -->
    <div
      v-if="isLoading"
      class="flex justify-center items-center py-12"
    >
      <UIcon
        name="i-heroicons-arrow-path"
        class="w-8 h-8 animate-spin text-primary"
      />
      <span class="ml-3 text-gray-600 dark:text-gray-400">Loading character...</span>
    </div>

    <!-- Error State -->
    <UAlert
      v-else-if="error"
      color="error"
      :title="error"
      class="mb-6"
    />

    <!-- Wizard Content -->
    <template v-else>
      <!-- Page Header -->
      <div class="text-center mb-6">
        <h1 class="text-3xl font-bold text-gray-900 dark:text-white">
          Edit Character
        </h1>
        <p class="mt-2 text-gray-600 dark:text-gray-400">
          {{ name || 'Continue building your hero' }}
        </p>
      </div>

      <!-- Compact Progress Bar -->
      <CharacterBuilderProgressBar
        :steps="activeSteps"
        :current-index="currentStepIndex"
        :current-label="currentStep?.label ?? ''"
        class="mb-8"
        @step-click="handleStepClick"
      />

      <!-- Step Content (nested route) -->
      <div class="bg-white dark:bg-gray-800 rounded-lg shadow-sm p-6">
        <NuxtPage />
      </div>

      <!-- Navigation Buttons -->
      <div class="flex justify-between mt-6">
        <UButton
          v-if="!isFirstStep"
          variant="outline"
          icon="i-heroicons-arrow-left"
          @click="previousStep()"
        >
          Back
        </UButton>
        <div v-else />

        <UButton
          v-if="!isLastStep"
          icon="i-heroicons-arrow-right"
          trailing
          @click="nextStep()"
        >
          Next
        </UButton>
      </div>
    </template>
  </div>
</template>
```

**Step 2: Run existing tests to check nothing broke**

Run: `docker compose exec nuxt npm run test 2>&1 | tail -20`

Expected: Tests should still pass (step pages not yet moved)

**Step 3: Commit**

```bash
git add app/pages/characters/[id]/edit.vue
git commit -m "refactor(wizard): convert edit.vue to layout wrapper with NuxtPage (#136)"
```

---

## Task 7: Create Step Page Files

**Files:**
- Create: `app/pages/characters/[id]/edit/name.vue`
- Create: `app/pages/characters/[id]/edit/race.vue`
- Create: `app/pages/characters/[id]/edit/class.vue`
- Create: `app/pages/characters/[id]/edit/abilities.vue`
- Create: `app/pages/characters/[id]/edit/background.vue`
- Create: `app/pages/characters/[id]/edit/proficiencies.vue`
- Create: `app/pages/characters/[id]/edit/equipment.vue`
- Create: `app/pages/characters/[id]/edit/spells.vue`
- Create: `app/pages/characters/[id]/edit/review.vue`

**Step 1: Create the edit directory**

```bash
mkdir -p app/pages/characters/\[id\]/edit
```

**Step 2: Create each step page (they wrap the existing components)**

Create `app/pages/characters/[id]/edit/name.vue`:

```vue
<!-- app/pages/characters/[id]/edit/name.vue -->
<template>
  <CharacterBuilderStepName />
</template>
```

Create `app/pages/characters/[id]/edit/race.vue`:

```vue
<!-- app/pages/characters/[id]/edit/race.vue -->
<template>
  <Suspense>
    <CharacterBuilderStepRace />
    <template #fallback>
      <div class="flex justify-center py-12">
        <UIcon
          name="i-heroicons-arrow-path"
          class="w-8 h-8 animate-spin text-primary"
        />
      </div>
    </template>
  </Suspense>
</template>
```

Create `app/pages/characters/[id]/edit/class.vue`:

```vue
<!-- app/pages/characters/[id]/edit/class.vue -->
<template>
  <Suspense>
    <CharacterBuilderStepClass />
    <template #fallback>
      <div class="flex justify-center py-12">
        <UIcon
          name="i-heroicons-arrow-path"
          class="w-8 h-8 animate-spin text-primary"
        />
      </div>
    </template>
  </Suspense>
</template>
```

Create `app/pages/characters/[id]/edit/abilities.vue`:

```vue
<!-- app/pages/characters/[id]/edit/abilities.vue -->
<template>
  <Suspense>
    <CharacterBuilderStepAbilities />
    <template #fallback>
      <div class="flex justify-center py-12">
        <UIcon
          name="i-heroicons-arrow-path"
          class="w-8 h-8 animate-spin text-primary"
        />
      </div>
    </template>
  </Suspense>
</template>
```

Create `app/pages/characters/[id]/edit/background.vue`:

```vue
<!-- app/pages/characters/[id]/edit/background.vue -->
<template>
  <Suspense>
    <CharacterBuilderStepBackground />
    <template #fallback>
      <div class="flex justify-center py-12">
        <UIcon
          name="i-heroicons-arrow-path"
          class="w-8 h-8 animate-spin text-primary"
        />
      </div>
    </template>
  </Suspense>
</template>
```

Create `app/pages/characters/[id]/edit/proficiencies.vue`:

```vue
<!-- app/pages/characters/[id]/edit/proficiencies.vue -->
<template>
  <Suspense>
    <CharacterBuilderStepProficiencies />
    <template #fallback>
      <div class="flex justify-center py-12">
        <UIcon
          name="i-heroicons-arrow-path"
          class="w-8 h-8 animate-spin text-primary"
        />
      </div>
    </template>
  </Suspense>
</template>
```

Create `app/pages/characters/[id]/edit/equipment.vue`:

```vue
<!-- app/pages/characters/[id]/edit/equipment.vue -->
<template>
  <Suspense>
    <CharacterBuilderStepEquipment />
    <template #fallback>
      <div class="flex justify-center py-12">
        <UIcon
          name="i-heroicons-arrow-path"
          class="w-8 h-8 animate-spin text-primary"
        />
      </div>
    </template>
  </Suspense>
</template>
```

Create `app/pages/characters/[id]/edit/spells.vue`:

```vue
<!-- app/pages/characters/[id]/edit/spells.vue -->
<template>
  <Suspense>
    <CharacterBuilderStepSpells />
    <template #fallback>
      <div class="flex justify-center py-12">
        <UIcon
          name="i-heroicons-arrow-path"
          class="w-8 h-8 animate-spin text-primary"
        />
      </div>
    </template>
  </Suspense>
</template>
```

Create `app/pages/characters/[id]/edit/review.vue`:

```vue
<!-- app/pages/characters/[id]/edit/review.vue -->
<template>
  <CharacterBuilderStepReview />
</template>
```

**Step 3: Commit**

```bash
git add app/pages/characters/\[id\]/edit/
git commit -m "feat(wizard): create route-based step pages (#136)"
```

---

## Task 8: Add Index Redirect

**Files:**
- Create: `app/pages/characters/[id]/edit/index.vue`

**Step 1: Create index redirect**

Create `app/pages/characters/[id]/edit/index.vue`:

```vue
<!-- app/pages/characters/[id]/edit/index.vue -->
<script setup lang="ts">
// Redirect to first step when accessing /edit without a step
const route = useRoute()
const characterId = route.params.id

await navigateTo(`/characters/${characterId}/edit/name`, { replace: true })
</script>
```

**Step 2: Commit**

```bash
git add app/pages/characters/\[id\]/edit/index.vue
git commit -m "feat(wizard): add index redirect to first step (#136)"
```

---

## Task 9: Remove Old Navigation from Store

**Files:**
- Modify: `app/stores/characterBuilder.ts`

**Step 1: Remove step navigation state and actions from store**

In `app/stores/characterBuilder.ts`, remove:

1. Remove the `currentStep` ref and `totalSteps` computed
2. Remove `isFirstStep` and `isLastStep` computed properties
3. Remove `nextStep()`, `previousStep()`, `goToStep()` functions
4. Remove these from the return statement
5. Keep the `determineStartingStep` function but have it return a step name instead of number

The store should no longer manage wizard navigation - that's now handled by the composable and routes.

**Step 2: Update any step components that call store.nextStep()**

Check each Step component - they should use the navigation composable instead:

```typescript
// OLD (in step components):
store.nextStep()

// NEW (in step components):
const { nextStep } = useWizardNavigation()
nextStep()
```

**Step 3: Run full test suite**

Run: `docker compose exec nuxt npm run test`

Fix any failing tests by updating them to use the new navigation pattern.

**Step 4: Commit**

```bash
git add app/stores/characterBuilder.ts app/components/character/builder/*.vue
git commit -m "refactor(wizard): remove navigation state from store, use composable (#136)"
```

---

## Task 10: Update Existing Tests

**Files:**
- Modify: `tests/stores/characterBuilder.test.ts`
- Modify: `tests/pages/characters/create.integration.test.ts`

**Step 1: Remove navigation tests from store tests**

In `tests/stores/characterBuilder.test.ts`, remove or update tests for:
- `currentStep`
- `totalSteps`
- `nextStep()`
- `previousStep()`
- `goToStep()`
- `isFirstStep`
- `isLastStep`

These are now tested in `tests/composables/useWizardSteps.test.ts`.

**Step 2: Update integration tests**

Update `tests/pages/characters/create.integration.test.ts` to test the new route-based navigation.

**Step 3: Run full test suite**

Run: `docker compose exec nuxt npm run test`

Expected: All tests pass

**Step 4: Commit**

```bash
git add tests/
git commit -m "test(wizard): update tests for route-based navigation (#136)"
```

---

## Task 11: Delete Old Stepper Component

**Files:**
- Delete: `app/components/character/builder/Stepper.vue`
- Delete: Test file if exists

**Step 1: Delete the old component**

```bash
rm app/components/character/builder/Stepper.vue
```

**Step 2: Verify no imports remain**

Run: `grep -r "CharacterBuilderStepper" app/`

Should return no results.

**Step 3: Run tests**

Run: `docker compose exec nuxt npm run test`

**Step 4: Commit**

```bash
git add -A
git commit -m "chore(wizard): remove old Stepper component (#136)"
```

---

## Task 12: Manual Testing & Final Verification

**Step 1: Start dev server**

```bash
docker compose exec nuxt npm run dev
```

**Step 2: Test wizard flow manually**

1. Go to `/characters` and create a new character
2. Verify URL changes to `/characters/[id]/edit/name`
3. Fill in name, click Next
4. Verify URL changes to `/characters/[id]/edit/race`
5. Continue through all steps
6. For a caster class, verify `/edit/spells` appears
7. For non-caster, verify `/edit/spells` is skipped
8. Click progress dots to navigate back
9. Verify can't click future dots
10. Complete wizard and verify Review step works

**Step 3: Run full test suite one more time**

```bash
docker compose exec nuxt npm run test
```

**Step 4: Run type check**

```bash
docker compose exec nuxt npm run typecheck
```

**Step 5: Run lint**

```bash
docker compose exec nuxt npm run lint:fix
```

**Step 6: Final commit if any fixes**

```bash
git add -A
git commit -m "fix(wizard): address issues found in manual testing (#136)"
```

---

## Summary

| Task | Description | Estimated Time |
|------|-------------|----------------|
| 1 | Create step registry type and composable | 10 min |
| 2 | Add conditional step visibility tests | 5 min |
| 3 | Create navigation composable | 15 min |
| 4 | Create compact progress bar component | 15 min |
| 5 | Create route middleware for step guards | 10 min |
| 6 | Create edit layout page | 10 min |
| 7 | Create step page files | 10 min |
| 8 | Add index redirect | 5 min |
| 9 | Remove old navigation from store | 20 min |
| 10 | Update existing tests | 20 min |
| 11 | Delete old stepper component | 5 min |
| 12 | Manual testing & verification | 15 min |

**Total: ~2.5 hours**

## Files Created/Modified Summary

**Created:**
- `app/composables/useWizardSteps.ts`
- `app/components/character/builder/ProgressBar.vue`
- `app/middleware/wizard-step.ts`
- `app/pages/characters/[id]/edit/name.vue`
- `app/pages/characters/[id]/edit/race.vue`
- `app/pages/characters/[id]/edit/class.vue`
- `app/pages/characters/[id]/edit/abilities.vue`
- `app/pages/characters/[id]/edit/background.vue`
- `app/pages/characters/[id]/edit/proficiencies.vue`
- `app/pages/characters/[id]/edit/equipment.vue`
- `app/pages/characters/[id]/edit/spells.vue`
- `app/pages/characters/[id]/edit/review.vue`
- `app/pages/characters/[id]/edit/index.vue`
- `tests/composables/useWizardSteps.test.ts`
- `tests/components/character/builder/ProgressBar.test.ts`
- `tests/middleware/wizard-step.test.ts`

**Modified:**
- `app/pages/characters/[id]/edit.vue`
- `app/stores/characterBuilder.ts`
- `tests/stores/characterBuilder.test.ts`
- `tests/pages/characters/create.integration.test.ts`

**Deleted:**
- `app/components/character/builder/Stepper.vue`
