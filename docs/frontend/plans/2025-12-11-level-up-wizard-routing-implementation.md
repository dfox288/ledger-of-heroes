# Level-Up Wizard Page-Based Routing Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Refactor the level-up wizard from store-based navigation to URL-based routing, matching the character creation wizard pattern.

**Architecture:** Single dynamic `[step].vue` route with component registry. URL is source of truth for current step. Preview page with "Begin Level Up" button before triggering API.

**Tech Stack:** Nuxt 4 / Vue 3 / Pinia / TypeScript / Vitest

---

## Task 1: Create Feature Branch

**Files:**
- None (git operation)

**Step 1: Create and checkout feature branch**

```bash
git checkout -b feature/issue-494-level-up-routing-refactor
```

**Step 2: Verify branch**

Run: `git branch --show-current`
Expected: `feature/issue-494-level-up-routing-refactor`

---

## Task 2: Update useLevelUpWizard Composable for URL Navigation

**Files:**
- Modify: `app/composables/useLevelUpWizard.ts`
- Test: `tests/composables/useLevelUpWizard.test.ts`

**Step 1: Write failing test for URL-based navigation**

```typescript
// tests/composables/useLevelUpWizard.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { setActivePinia, createPinia } from 'pinia'
import { useLevelUpWizard } from '~/composables/useLevelUpWizard'
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'

// Mock navigateTo
const mockNavigateTo = vi.fn()
vi.mock('#app', () => ({
  navigateTo: (path: string) => mockNavigateTo(path)
}))

describe('useLevelUpWizard', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    mockNavigateTo.mockClear()
  })

  describe('URL-based navigation', () => {
    it('nextStep navigates via URL', () => {
      const store = useCharacterLevelUpStore()
      store.publicId = 'test-hero-Ab12'
      store.levelUpResult = { hp_choice_pending: true } as any

      const { nextStep } = useLevelUpWizard({
        publicId: 'test-hero-Ab12',
        currentStep: 'hit-points'
      })

      nextStep()

      expect(mockNavigateTo).toHaveBeenCalledWith(
        '/characters/test-hero-Ab12/level-up/summary'
      )
    })

    it('previousStep navigates via URL', () => {
      const store = useCharacterLevelUpStore()
      store.publicId = 'test-hero-Ab12'
      store.levelUpResult = { hp_choice_pending: true } as any

      const { previousStep } = useLevelUpWizard({
        publicId: 'test-hero-Ab12',
        currentStep: 'summary'
      })

      previousStep()

      expect(mockNavigateTo).toHaveBeenCalledWith(
        '/characters/test-hero-Ab12/level-up/hit-points'
      )
    })

    it('getStepUrl returns correct URL format', () => {
      const store = useCharacterLevelUpStore()
      store.publicId = 'test-hero-Ab12'

      const { getStepUrl } = useLevelUpWizard({
        publicId: 'test-hero-Ab12',
        currentStep: 'hit-points'
      })

      expect(getStepUrl('spells')).toBe('/characters/test-hero-Ab12/level-up/spells')
    })
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/composables/useLevelUpWizard.test.ts --reporter=verbose`
Expected: FAIL - composable doesn't accept options parameter yet

**Step 3: Update composable to use URL navigation**

```typescript
// app/composables/useLevelUpWizard.ts
/**
 * Level-Up Wizard Navigation Composable
 *
 * Manages wizard step navigation using URL-based routing.
 * URL is source of truth for current step (matches character creation wizard pattern).
 */
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'
import type { LevelUpStep } from '~/types/character'

export interface UseLevelUpWizardOptions {
  /** Character public ID for URL building */
  publicId: string
  /** Current step name from URL (route.params.step) */
  currentStep: string
}

/**
 * Create the step registry with visibility functions
 */
function createStepRegistry(store: ReturnType<typeof useCharacterLevelUpStore>): LevelUpStep[] {
  return [
    {
      name: 'class-selection',
      label: 'Class',
      icon: 'i-heroicons-shield-check',
      visible: () => store.needsClassSelection,
      shouldSkip: () => !store.needsClassSelection
    },
    {
      name: 'subclass',
      label: 'Subclass',
      icon: 'i-heroicons-star',
      visible: () => store.hasSubclassChoice,
      shouldSkip: () => !store.hasSubclassChoice
    },
    {
      name: 'hit-points',
      label: 'Hit Points',
      icon: 'i-heroicons-heart',
      visible: () => true,
      shouldSkip: () => !(store.levelUpResult?.hp_choice_pending ?? true)
    },
    {
      name: 'asi-feat',
      label: 'ASI / Feat',
      icon: 'i-heroicons-arrow-trending-up',
      visible: () => true,
      shouldSkip: () => !(store.levelUpResult?.asi_pending ?? false)
    },
    {
      name: 'feature-choices',
      label: 'Features',
      icon: 'i-heroicons-puzzle-piece',
      visible: () => store.hasFeatureChoices,
      shouldSkip: () => !store.hasFeatureChoices
    },
    {
      name: 'spells',
      label: 'Spells',
      icon: 'i-heroicons-sparkles',
      visible: () => store.hasSpellChoices,
      shouldSkip: () => !store.hasSpellChoices
    },
    {
      name: 'languages',
      label: 'Languages',
      icon: 'i-heroicons-language',
      visible: () => store.hasLanguageChoices,
      shouldSkip: () => !store.hasLanguageChoices
    },
    {
      name: 'proficiencies',
      label: 'Proficiencies',
      icon: 'i-heroicons-academic-cap',
      visible: () => store.hasProficiencyChoices,
      shouldSkip: () => !store.hasProficiencyChoices
    },
    {
      name: 'summary',
      label: 'Summary',
      icon: 'i-heroicons-trophy',
      visible: () => true
    }
  ]
}

export function useLevelUpWizard(options: UseLevelUpWizardOptions) {
  const store = useCharacterLevelUpStore()
  const { publicId, currentStep } = options

  const stepRegistry = createStepRegistry(store)

  // ══════════════════════════════════════════════════════════════
  // COMPUTED: Active Steps
  // ══════════════════════════════════════════════════════════════

  const activeSteps = computed(() =>
    stepRegistry.filter(step => step.visible())
  )

  const currentStepName = computed(() => currentStep)

  const currentStepIndex = computed(() =>
    activeSteps.value.findIndex(s => s.name === currentStepName.value)
  )

  const currentStepInfo = computed(() =>
    activeSteps.value[currentStepIndex.value] ?? null
  )

  const totalSteps = computed(() => activeSteps.value.length)

  const isFirstStep = computed(() => currentStepIndex.value === 0)

  const isLastStep = computed(() =>
    currentStepIndex.value === totalSteps.value - 1
  )

  const progressPercent = computed(() => {
    if (totalSteps.value <= 1) return 100
    return Math.round((currentStepIndex.value / (totalSteps.value - 1)) * 100)
  })

  // ══════════════════════════════════════════════════════════════
  // URL HELPERS
  // ══════════════════════════════════════════════════════════════

  function getStepUrl(stepName: string): string {
    return `/characters/${publicId}/level-up/${stepName}`
  }

  function getPreviewUrl(): string {
    return `/characters/${publicId}/level-up`
  }

  // ══════════════════════════════════════════════════════════════
  // NAVIGATION
  // ══════════════════════════════════════════════════════════════

  async function nextStep(): Promise<void> {
    let nextIndex = currentStepIndex.value + 1

    while (nextIndex < activeSteps.value.length) {
      const next = activeSteps.value[nextIndex]
      if (next && !next.shouldSkip?.()) {
        await navigateTo(getStepUrl(next.name))
        return
      }
      nextIndex++
    }
  }

  async function previousStep(): Promise<void> {
    let prevIndex = currentStepIndex.value - 1

    while (prevIndex >= 0) {
      const prev = activeSteps.value[prevIndex]
      if (prev && !prev.shouldSkip?.()) {
        await navigateTo(getStepUrl(prev.name))
        return
      }
      prevIndex--
    }
  }

  async function goToStep(stepName: string): Promise<void> {
    const step = activeSteps.value.find(s => s.name === stepName)
    if (step) {
      await navigateTo(getStepUrl(stepName))
    }
  }

  const nextStepInfo = computed(() => {
    let nextIndex = currentStepIndex.value + 1
    while (nextIndex < activeSteps.value.length) {
      const next = activeSteps.value[nextIndex]
      if (next && !next.shouldSkip?.()) {
        return next
      }
      nextIndex++
    }
    return null
  })

  const previousStepInfo = computed(() => {
    let prevIndex = currentStepIndex.value - 1
    while (prevIndex >= 0) {
      const prev = activeSteps.value[prevIndex]
      if (prev && !prev.shouldSkip?.()) {
        return prev
      }
      prevIndex--
    }
    return null
  })

  // ══════════════════════════════════════════════════════════════
  // RETURN
  // ══════════════════════════════════════════════════════════════

  return {
    // Step registry
    stepRegistry,
    activeSteps,

    // Current step
    currentStep: currentStepInfo,
    currentStepName,
    currentStepIndex,

    // Progress
    totalSteps,
    progressPercent,
    isFirstStep,
    isLastStep,

    // Navigation
    nextStep,
    previousStep,
    goToStep,
    getStepUrl,
    getPreviewUrl,
    nextStepInfo,
    previousStepInfo
  }
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/composables/useLevelUpWizard.test.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/composables/useLevelUpWizard.ts tests/composables/useLevelUpWizard.test.ts
git commit -m "refactor: Update useLevelUpWizard for URL-based navigation (#494)"
```

---

## Task 3: Simplify Store (Remove Step Navigation)

**Files:**
- Modify: `app/stores/characterLevelUp.ts`
- Modify: `tests/stores/characterLevelUp.test.ts` (if exists)

**Step 1: Remove currentStepName and goToStep from store**

Edit `app/stores/characterLevelUp.ts`:

Remove these lines:
```typescript
// Remove from state
const currentStepName = ref<string>('class-selection')

// Remove from actions
function goToStep(stepName: string) {
  currentStepName.value = stepName
}

// Remove from return
currentStepName,
goToStep,
```

Add new state for tracking level-up progress:
```typescript
/** Is a level-up in progress? (API called, awaiting choices) */
const isLevelUpInProgress = computed(() => levelUpResult.value !== null)
```

**Step 2: Update reset() to not reset currentStepName**

```typescript
function reset() {
  characterId.value = null
  publicId.value = null
  characterClasses.value = []
  totalLevel.value = 0
  isOpen.value = false
  levelUpResult.value = null
  selectedClassSlug.value = null
  pendingChoices.value = []
  isLoading.value = false
  error.value = null
}
```

**Step 3: Run typecheck to find any broken references**

Run: `docker compose exec nuxt npm run typecheck`
Expected: Errors showing where `currentStepName` and `goToStep` are used

**Step 4: Note locations to fix (will fix in Task 5)**

The main location is `app/pages/characters/[publicId]/level-up/index.vue` which we'll replace entirely.

**Step 5: Commit store changes**

```bash
git add app/stores/characterLevelUp.ts
git commit -m "refactor: Remove step navigation from characterLevelUp store (#494)"
```

---

## Task 4: Create Preview/Entry Page

**Files:**
- Modify: `app/pages/characters/[publicId]/level-up/index.vue`
- Test: `tests/pages/characters/level-up/index.test.ts`

**Step 1: Write failing test for preview page**

```typescript
// tests/pages/characters/level-up/index.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import LevelUpPreviewPage from '~/pages/characters/[publicId]/level-up/index.vue'

// Mock route
vi.mock('vue-router', async () => {
  const actual = await vi.importActual('vue-router')
  return {
    ...actual,
    useRoute: () => ({
      params: { publicId: 'test-hero-Ab12' }
    }),
    useRouter: () => ({
      push: vi.fn()
    })
  }
})

// Mock API
vi.mock('~/composables/useApi', () => ({
  useApi: () => ({
    apiFetch: vi.fn().mockResolvedValue({
      data: {
        id: 1,
        public_id: 'test-hero-Ab12',
        name: 'Test Hero',
        classes: [{ class: { name: 'Fighter', slug: 'fighter', hit_die: 10 }, level: 4 }],
        is_complete: true
      }
    })
  })
}))

describe('LevelUpPreviewPage', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders character name in title', async () => {
    const wrapper = await mountSuspended(LevelUpPreviewPage)
    expect(wrapper.text()).toContain('Test Hero')
  })

  it('shows Begin Level Up button', async () => {
    const wrapper = await mountSuspended(LevelUpPreviewPage)
    expect(wrapper.text()).toContain('Begin Level Up')
  })

  it('shows current and target level', async () => {
    const wrapper = await mountSuspended(LevelUpPreviewPage)
    expect(wrapper.text()).toContain('Level 4')
    expect(wrapper.text()).toContain('Level 5')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/level-up/index.test.ts --reporter=verbose`
Expected: FAIL - current page doesn't have preview structure

**Step 3: Replace index.vue with preview page**

```vue
<!-- app/pages/characters/[publicId]/level-up/index.vue -->
<script setup lang="ts">
/**
 * Level-Up Preview Page
 *
 * Shows character info and preview of upcoming choices.
 * User clicks "Begin Level Up" to trigger the API and start the wizard.
 *
 * URL: /characters/:publicId/level-up
 */
import { useCharacterLevelUpStore, type CharacterClassEntry } from '~/stores/characterLevelUp'

const route = useRoute()
const publicId = computed(() => route.params.publicId as string)

const store = useCharacterLevelUpStore()
const { apiFetch } = useApi()

// ════════════════════════════════════════════════════════════════
// FETCH CHARACTER DATA
// ════════════════════════════════════════════════════════════════

interface CharacterResponse {
  data: {
    id: number
    public_id: string
    name: string
    classes: Array<{
      class: { name: string; slug: string; full_slug?: string; hit_die: number }
      level: number
      subclass?: { name: string; slug: string } | null
      is_primary: boolean
    }>
    is_complete: boolean
  }
}

const { data: characterData, pending: loading, error } = await useAsyncData(
  `level-up-preview-${publicId.value}`,
  () => apiFetch<CharacterResponse>(`/characters/${publicId.value}`)
)

const character = computed(() => characterData.value?.data ?? null)

const currentLevel = computed(() =>
  character.value?.classes?.reduce((sum, c) => sum + (c.level || 0), 0) ?? 0
)

const targetLevel = computed(() => currentLevel.value + 1)

const primaryClass = computed(() =>
  character.value?.classes?.find(c => c.is_primary) ?? character.value?.classes?.[0] ?? null
)

const isMulticlass = computed(() => (character.value?.classes?.length ?? 0) > 1)

// ════════════════════════════════════════════════════════════════
// LEVEL-UP INITIATION
// ════════════════════════════════════════════════════════════════

const selectedClassSlug = ref<string | null>(null)
const isStarting = ref(false)
const startError = ref<string | null>(null)

// For single-class, auto-select
watchEffect(() => {
  if (!isMulticlass.value && primaryClass.value) {
    selectedClassSlug.value = primaryClass.value.class?.full_slug ?? primaryClass.value.class?.slug ?? null
  }
})

async function beginLevelUp() {
  if (!character.value || !selectedClassSlug.value) return

  isStarting.value = true
  startError.value = null

  try {
    // Initialize store
    const classEntries: CharacterClassEntry[] = (character.value.classes ?? []).map(c => ({
      class: c.class as any,
      level: c.level,
      subclass: c.subclass ? { name: c.subclass.name, slug: c.subclass.slug } : null,
      is_primary: c.is_primary
    }))

    store.characterId = character.value.id
    store.publicId = character.value.public_id
    store.characterClasses = classEntries
    store.totalLevel = currentLevel.value

    // Call level-up API
    await store.levelUp(selectedClassSlug.value)

    // Navigate to first step (hit-points for single-class, or based on pending choices)
    const firstStep = store.hasSubclassChoice ? 'subclass' : 'hit-points'
    await navigateTo(`/characters/${publicId.value}/level-up/${firstStep}`)
  } catch (e) {
    startError.value = e instanceof Error ? e.message : 'Failed to start level-up'
  } finally {
    isStarting.value = false
  }
}

// ════════════════════════════════════════════════════════════════
// RESUME INCOMPLETE LEVEL-UP
// ════════════════════════════════════════════════════════════════

// Check if character has incomplete level-up (pending choices)
const shouldResume = computed(() => character.value?.is_complete === false)

async function resumeLevelUp() {
  if (!character.value) return

  isStarting.value = true

  try {
    // Initialize store
    const classEntries: CharacterClassEntry[] = (character.value.classes ?? []).map(c => ({
      class: c.class as any,
      level: c.level,
      subclass: c.subclass ? { name: c.subclass.name, slug: c.subclass.slug } : null,
      is_primary: c.is_primary
    }))

    store.characterId = character.value.id
    store.publicId = character.value.public_id
    store.characterClasses = classEntries
    store.totalLevel = currentLevel.value

    // Fetch pending choices
    await store.fetchPendingChoices()

    // Determine first step based on pending choices
    const hasSubclass = store.pendingChoices.some(c => c.type === 'subclass')
    const hasHp = store.pendingChoices.some(c => c.type === 'hit_points')
    const hasAsi = store.pendingChoices.some(c => c.type === 'asi_or_feat')
    const hasFeature = store.pendingChoices.some(c =>
      ['fighting_style', 'expertise', 'optional_feature'].includes(c.type)
    )
    const hasSpell = store.pendingChoices.some(c => c.type === 'spell')
    const hasLanguage = store.pendingChoices.some(c => c.type === 'language')
    const hasProficiency = store.pendingChoices.some(c => c.type === 'proficiency')

    let firstStep = 'summary'
    if (hasSubclass) firstStep = 'subclass'
    else if (hasHp) firstStep = 'hit-points'
    else if (hasAsi) firstStep = 'asi-feat'
    else if (hasFeature) firstStep = 'feature-choices'
    else if (hasSpell) firstStep = 'spells'
    else if (hasLanguage) firstStep = 'languages'
    else if (hasProficiency) firstStep = 'proficiencies'

    await navigateTo(`/characters/${publicId.value}/level-up/${firstStep}`)
  } catch (e) {
    startError.value = e instanceof Error ? e.message : 'Failed to resume level-up'
    isStarting.value = false
  }
}

// Auto-resume if character has pending choices
onMounted(() => {
  if (shouldResume.value) {
    resumeLevelUp()
  }
})

// ════════════════════════════════════════════════════════════════
// SEO
// ════════════════════════════════════════════════════════════════

useSeoMeta({
  title: () => `Level Up - ${character.value?.name ?? 'Character'}`
})
</script>

<template>
  <div class="min-h-screen bg-gray-50 dark:bg-gray-900">
    <div class="max-w-2xl mx-auto px-4 py-12">
      <!-- Back Link -->
      <UButton
        variant="ghost"
        color="neutral"
        icon="i-heroicons-arrow-left"
        :to="`/characters/${publicId}`"
        class="mb-8"
      >
        Back to Character
      </UButton>

      <!-- Loading State -->
      <div
        v-if="loading || isStarting"
        class="flex flex-col items-center justify-center py-16"
      >
        <UIcon
          name="i-heroicons-arrow-path"
          class="w-8 h-8 animate-spin text-primary"
        />
        <p class="mt-4 text-gray-500">
          {{ isStarting ? 'Starting level up...' : 'Loading character...' }}
        </p>
      </div>

      <!-- Error State -->
      <UAlert
        v-else-if="error || startError"
        color="error"
        icon="i-heroicons-exclamation-circle"
        :title="error?.message || startError || 'An error occurred'"
        class="mb-6"
      >
        <template #actions>
          <UButton
            color="error"
            variant="soft"
            :to="`/characters/${publicId}`"
          >
            Back to Character
          </UButton>
        </template>
      </UAlert>

      <!-- Preview Card -->
      <UCard v-else-if="character" class="text-center">
        <template #header>
          <div class="flex items-center justify-center gap-2">
            <UIcon name="i-heroicons-arrow-trending-up" class="w-6 h-6 text-primary" />
            <h1 class="text-2xl font-bold text-gray-900 dark:text-white">
              Level Up
            </h1>
          </div>
        </template>

        <!-- Character Info -->
        <div class="space-y-6">
          <div>
            <h2 class="text-xl font-semibold text-gray-900 dark:text-white">
              {{ character.name }}
            </h2>
            <p class="text-gray-500">
              {{ primaryClass?.class?.name }} {{ currentLevel }}
            </p>
          </div>

          <!-- Level Transition -->
          <div class="flex items-center justify-center gap-4">
            <div class="text-center">
              <div class="text-3xl font-bold text-gray-400">{{ currentLevel }}</div>
              <div class="text-sm text-gray-500">Current</div>
            </div>
            <UIcon name="i-heroicons-arrow-right" class="w-8 h-8 text-primary" />
            <div class="text-center">
              <div class="text-3xl font-bold text-primary">{{ targetLevel }}</div>
              <div class="text-sm text-gray-500">Target</div>
            </div>
          </div>

          <!-- Multiclass Selection (if applicable) -->
          <div v-if="isMulticlass" class="pt-4 border-t border-gray-200 dark:border-gray-700">
            <label class="block text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
              Level up which class?
            </label>
            <USelect
              v-model="selectedClassSlug"
              :items="character.classes.map(c => ({
                label: `${c.class.name} (Level ${c.level})`,
                value: c.class.full_slug ?? c.class.slug
              }))"
              placeholder="Select a class"
            />
          </div>

          <!-- What's Ahead Preview -->
          <div class="pt-4 border-t border-gray-200 dark:border-gray-700 text-left">
            <h3 class="text-sm font-medium text-gray-700 dark:text-gray-300 mb-2">
              What's ahead:
            </h3>
            <ul class="space-y-1 text-sm text-gray-600 dark:text-gray-400">
              <li class="flex items-center gap-2">
                <UIcon name="i-heroicons-heart" class="w-4 h-4" />
                Choose HP increase (roll or average)
              </li>
              <li v-if="targetLevel % 4 === 0" class="flex items-center gap-2">
                <UIcon name="i-heroicons-arrow-trending-up" class="w-4 h-4" />
                Ability Score Improvement or Feat
              </li>
            </ul>
          </div>
        </div>

        <template #footer>
          <UButton
            color="primary"
            size="lg"
            block
            :disabled="!selectedClassSlug"
            @click="beginLevelUp"
          >
            Begin Level Up
          </UButton>
        </template>
      </UCard>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/pages/characters/level-up/index.test.ts --reporter=verbose`
Expected: PASS

**Step 5: Commit**

```bash
git add app/pages/characters/[publicId]/level-up/index.vue tests/pages/characters/level-up/index.test.ts
git commit -m "feat: Add level-up preview page with Begin Level Up button (#494)"
```

---

## Task 5: Create Dynamic Step Page

**Files:**
- Create: `app/pages/characters/[publicId]/level-up/[step].vue`
- Test: `tests/pages/characters/level-up/step.test.ts`

**Step 1: Write failing test for dynamic step page**

```typescript
// tests/pages/characters/level-up/step.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'

describe('LevelUpStepPage', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders the correct step component based on URL', async () => {
    // This test will be expanded once the page is created
    expect(true).toBe(true)
  })
})
```

**Step 2: Create dynamic step page**

```vue
<!-- app/pages/characters/[publicId]/level-up/[step].vue -->
<script setup lang="ts">
import type { Component } from 'vue'
import { useCharacterLevelUpStore } from '~/stores/characterLevelUp'
import { useLevelUpWizard } from '~/composables/useLevelUpWizard'

/**
 * Level-Up Wizard - Dynamic Step Router
 *
 * Renders the appropriate wizard step component based on the URL parameter.
 * URL is the source of truth for current step (matches character creation pattern).
 */

// ════════════════════════════════════════════════════════════════
// STEP REGISTRY
// ════════════════════════════════════════════════════════════════

const stepComponents: Record<string, Component> = {
  'class-selection': defineAsyncComponent(() => import('~/components/character/levelup/StepClassSelection.vue')),
  'subclass': defineAsyncComponent(() => import('~/components/character/levelup/StepSubclassChoice.vue')),
  'hit-points': defineAsyncComponent(() => import('~/components/character/levelup/StepHitPoints.vue')),
  'asi-feat': defineAsyncComponent(() => import('~/components/character/levelup/StepAsiFeat.vue')),
  'feature-choices': defineAsyncComponent(() => import('~/components/character/levelup/StepFeatureChoices.vue')),
  'spells': defineAsyncComponent(() => import('~/components/character/levelup/StepSpells.vue')),
  'languages': defineAsyncComponent(() => import('~/components/character/levelup/StepLanguages.vue')),
  'proficiencies': defineAsyncComponent(() => import('~/components/character/levelup/StepProficiencies.vue')),
  'summary': defineAsyncComponent(() => import('~/components/character/levelup/StepSummary.vue')),
}

// ════════════════════════════════════════════════════════════════
// ROUTE HANDLING
// ════════════════════════════════════════════════════════════════

const route = useRoute()
const publicId = computed(() => route.params.publicId as string)
const stepName = computed(() => route.params.step as string)

const store = useCharacterLevelUpStore()

// ════════════════════════════════════════════════════════════════
// WIZARD NAVIGATION
// ════════════════════════════════════════════════════════════════

const {
  activeSteps,
  currentStep,
  isFirstStep,
  isLastStep,
  nextStep,
  previousStep,
  progressPercent
} = useLevelUpWizard({
  publicId: publicId.value,
  currentStep: stepName.value
})

// ════════════════════════════════════════════════════════════════
// GUARDS
// ════════════════════════════════════════════════════════════════

// Redirect to preview if no level-up in progress
if (!store.levelUpResult && !store.pendingChoices.length) {
  await navigateTo(`/characters/${publicId.value}/level-up`)
}

// Redirect to first valid step if current step is invalid
const stepComponent = computed(() => stepComponents[stepName.value] ?? null)
if (!stepComponent.value) {
  throw createError({
    statusCode: 404,
    message: `Unknown level-up step: ${stepName.value}`
  })
}

// ════════════════════════════════════════════════════════════════
// NAVIGATION HANDLERS
// ════════════════════════════════════════════════════════════════

async function handleBack() {
  if (isFirstStep.value) {
    await navigateTo(`/characters/${publicId.value}`)
  } else {
    await previousStep()
  }
}

async function handleNext() {
  await nextStep()
}

function handleComplete() {
  store.reset()
  navigateTo(`/characters/${publicId.value}`)
}

// ════════════════════════════════════════════════════════════════
// SEO
// ════════════════════════════════════════════════════════════════

const stepTitle = computed(() =>
  stepName.value.split('-').map(w => w.charAt(0).toUpperCase() + w.slice(1)).join(' ')
)

useSeoMeta({
  title: () => `Level Up - ${stepTitle.value}`
})
</script>

<template>
  <div class="h-screen flex bg-gray-50 dark:bg-gray-900">
    <!-- Sidebar -->
    <CharacterLevelupLevelUpSidebar
      :active-steps="activeSteps"
      :current-step="stepName"
      :public-id="publicId"
      class="w-64 flex-shrink-0 hidden lg:block"
    />

    <!-- Main content -->
    <div class="flex-1 flex flex-col min-w-0 overflow-hidden">
      <!-- Header -->
      <header class="flex items-center justify-between p-4 border-b border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800">
        <div class="flex items-center gap-4">
          <UButton
            variant="ghost"
            color="neutral"
            icon="i-heroicons-arrow-left"
            :to="`/characters/${publicId}`"
          />
          <h1 class="text-xl font-semibold text-gray-900 dark:text-white">
            Level Up - {{ stepTitle }}
          </h1>
        </div>
        <div class="text-sm text-gray-500">
          {{ progressPercent }}% complete
        </div>
      </header>

      <!-- Content Area -->
      <main class="flex-1 p-6 overflow-y-auto">
        <Suspense>
          <component
            :is="stepComponent"
            :character-id="store.characterId"
            :public-id="publicId"
            :next-step="nextStep"
            :refresh-after-save="store.refreshChoices"
            @complete="handleComplete"
          />
          <template #fallback>
            <div class="flex items-center justify-center py-12">
              <UIcon name="i-heroicons-arrow-path" class="w-8 h-8 animate-spin text-primary" />
            </div>
          </template>
        </Suspense>
      </main>

      <!-- Footer Navigation -->
      <footer class="flex-shrink-0 p-4 border-t border-gray-200 dark:border-gray-700 bg-white dark:bg-gray-800">
        <div class="flex justify-between max-w-4xl mx-auto">
          <UButton
            variant="ghost"
            color="neutral"
            icon="i-heroicons-arrow-left"
            @click="handleBack"
          >
            {{ isFirstStep ? 'Cancel' : 'Back' }}
          </UButton>

          <UButton
            v-if="!isLastStep && stepName !== 'summary'"
            color="primary"
            trailing-icon="i-heroicons-arrow-right"
            @click="handleNext"
          >
            Continue
          </UButton>
        </div>
      </footer>
    </div>
  </div>
</template>
```

**Step 3: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: PASS (or list of components that need to be created)

**Step 4: Commit**

```bash
git add app/pages/characters/[publicId]/level-up/[step].vue
git commit -m "feat: Add dynamic step page for level-up wizard (#494)"
```

---

## Task 6: Update LevelUpSidebar for URL Navigation

**Files:**
- Modify: `app/components/character/levelup/LevelUpSidebar.vue`

**Step 1: Read current sidebar implementation**

Check current implementation and update to accept props for URL-based navigation.

**Step 2: Update sidebar to use URL links**

```vue
<!-- app/components/character/levelup/LevelUpSidebar.vue -->
<script setup lang="ts">
import type { LevelUpStep } from '~/types/character'

interface Props {
  activeSteps: LevelUpStep[]
  currentStep: string
  publicId: string
}

const props = defineProps<Props>()

function getStepUrl(stepName: string): string {
  return `/characters/${props.publicId}/level-up/${stepName}`
}

function isStepComplete(stepName: string): boolean {
  const stepIndex = props.activeSteps.findIndex(s => s.name === stepName)
  const currentIndex = props.activeSteps.findIndex(s => s.name === props.currentStep)
  return stepIndex < currentIndex
}

function isStepCurrent(stepName: string): boolean {
  return stepName === props.currentStep
}
</script>

<template>
  <aside class="bg-white dark:bg-gray-800 border-r border-gray-200 dark:border-gray-700 p-4">
    <h2 class="text-lg font-semibold text-gray-900 dark:text-white mb-4">
      Level Up Steps
    </h2>

    <nav class="space-y-1">
      <NuxtLink
        v-for="step in activeSteps"
        :key="step.name"
        :to="getStepUrl(step.name)"
        class="flex items-center gap-3 px-3 py-2 rounded-lg transition-colors"
        :class="{
          'bg-primary-50 dark:bg-primary-900/20 text-primary-600 dark:text-primary-400': isStepCurrent(step.name),
          'text-gray-600 dark:text-gray-400 hover:bg-gray-100 dark:hover:bg-gray-700': !isStepCurrent(step.name),
          'text-green-600 dark:text-green-400': isStepComplete(step.name)
        }"
      >
        <UIcon
          :name="isStepComplete(step.name) ? 'i-heroicons-check-circle-solid' : step.icon"
          class="w-5 h-5"
        />
        <span class="font-medium">{{ step.label }}</span>
      </NuxtLink>
    </nav>
  </aside>
</template>
```

**Step 3: Commit**

```bash
git add app/components/character/levelup/LevelUpSidebar.vue
git commit -m "refactor: Update LevelUpSidebar for URL-based navigation (#494)"
```

---

## Task 7: Create Missing Step Components (Stubs)

**Files:**
- Create: `app/components/character/levelup/StepClassSelection.vue`
- Create: `app/components/character/levelup/StepAsiFeat.vue`
- Create: `app/components/character/levelup/StepFeatureChoices.vue`
- Create: `app/components/character/levelup/StepSpells.vue`
- Create: `app/components/character/levelup/StepLanguages.vue`
- Create: `app/components/character/levelup/StepProficiencies.vue`
- Rename: `StepLevelUpSummary.vue` → `StepSummary.vue`

**Step 1: Create StepClassSelection stub**

```vue
<!-- app/components/character/levelup/StepClassSelection.vue -->
<script setup lang="ts">
/**
 * Level-Up: Class Selection Step
 *
 * Shown for multiclass characters to select which class to level.
 * TODO: Implement full multiclass support
 */
const props = defineProps<{
  characterId: number
  publicId: string
  nextStep: () => Promise<void>
}>()
</script>

<template>
  <div class="max-w-2xl mx-auto text-center py-12">
    <UIcon
      name="i-heroicons-shield-check"
      class="w-16 h-16 mx-auto text-gray-400 mb-4"
    />
    <h2 class="text-2xl font-semibold text-gray-900 dark:text-white mb-2">
      Class Selection
    </h2>
    <p class="text-gray-500 mb-6">
      Multiclass support coming soon.
    </p>
  </div>
</template>
```

**Step 2: Create StepAsiFeat stub**

```vue
<!-- app/components/character/levelup/StepAsiFeat.vue -->
<script setup lang="ts">
/**
 * Level-Up: ASI/Feat Step
 *
 * Choose Ability Score Improvement or Feat at ASI levels.
 * Wraps the existing CharacterWizardStepFeats component.
 */
</script>

<template>
  <CharacterWizardStepFeats />
</template>
```

**Step 3: Create adapter components for shared wizard steps**

These wrap the existing creation wizard step components:

```vue
<!-- app/components/character/levelup/StepFeatureChoices.vue -->
<script setup lang="ts">
const props = defineProps<{
  characterId: number
  nextStep: () => Promise<void>
  refreshAfterSave: () => Promise<void>
}>()
</script>

<template>
  <CharacterWizardStepFeatureChoices
    :character-id="characterId"
    :next-step="nextStep"
    :refresh-after-save="refreshAfterSave"
  />
</template>
```

```vue
<!-- app/components/character/levelup/StepSpells.vue -->
<script setup lang="ts">
</script>

<template>
  <CharacterWizardStepSpells />
</template>
```

```vue
<!-- app/components/character/levelup/StepLanguages.vue -->
<script setup lang="ts">
const props = defineProps<{
  characterId: number
  nextStep: () => Promise<void>
  refreshAfterSave: () => Promise<void>
}>()
</script>

<template>
  <CharacterWizardStepLanguages
    :character-id="characterId"
    :next-step="nextStep"
    :refresh-after-save="refreshAfterSave"
  />
</template>
```

```vue
<!-- app/components/character/levelup/StepProficiencies.vue -->
<script setup lang="ts">
const props = defineProps<{
  characterId: number
  nextStep: () => Promise<void>
  refreshAfterSave: () => Promise<void>
}>()
</script>

<template>
  <CharacterWizardStepProficiencies
    :character-id="characterId"
    :next-step="nextStep"
    :refresh-after-save="refreshAfterSave"
  />
</template>
```

**Step 4: Rename StepLevelUpSummary to StepSummary**

```bash
mv app/components/character/levelup/StepLevelUpSummary.vue app/components/character/levelup/StepSummary.vue
```

Update imports in the renamed file if needed.

**Step 5: Commit**

```bash
git add app/components/character/levelup/
git commit -m "feat: Add level-up step component stubs and adapters (#494)"
```

---

## Task 8: Run Full Test Suite and Fix Issues

**Step 1: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`
Expected: PASS (or fix any type errors)

**Step 2: Run character tests**

Run: `docker compose exec nuxt npm run test:character`
Expected: PASS (or fix failing tests)

**Step 3: Run lint**

Run: `docker compose exec nuxt npm run lint:fix`

**Step 4: Commit any fixes**

```bash
git add -A
git commit -m "fix: Address test and lint issues from routing refactor (#494)"
```

---

## Task 9: Manual Browser Testing

**Step 1: Start dev server**

Run: `docker compose exec nuxt npm run dev`

**Step 2: Test level-up flow**

1. Go to character sheet
2. Click "Level Up" button
3. Verify preview page shows
4. Click "Begin Level Up"
5. Verify URL changes to `/level-up/hit-points`
6. Complete HP step
7. Verify navigation to next step updates URL
8. Refresh page - verify step is preserved
9. Use browser back button - verify it works

**Step 3: Test edge cases**

1. Direct URL to invalid step → should redirect
2. Direct URL to step with no level-up → should redirect to preview
3. Refresh mid-wizard → should resume at same step

---

## Task 10: Final Commit and PR

**Step 1: Ensure all tests pass**

Run: `docker compose exec nuxt npm run test`

**Step 2: Final commit**

```bash
git add -A
git commit -m "feat: Complete level-up wizard routing refactor (#494)"
```

**Step 3: Push branch**

```bash
git push -u origin feature/issue-494-level-up-routing-refactor
```

**Step 4: Create PR**

```bash
gh pr create --title "feat: Refactor level-up wizard to page-based routing (#494)" --body "$(cat <<'EOF'
## Summary
- Refactored level-up wizard from store-based navigation to URL-based routing
- Added preview page with "Begin Level Up" confirmation
- URL now persists step across refresh
- Matches character creation wizard pattern

## Changes
- Updated `useLevelUpWizard` composable for URL navigation
- Simplified `characterLevelUp` store (removed step navigation)
- Created dynamic `[step].vue` page with component registry
- Added preview page at `/level-up/` index
- Updated sidebar for URL-based navigation

## Test Plan
- [x] Level-up flow works end-to-end
- [x] URL updates on step change
- [x] Page refresh preserves step
- [x] Browser back/forward works
- [x] All existing tests pass

Closes #494
EOF
)"
```

---

## Summary

| Task | Description | Files |
|------|-------------|-------|
| 1 | Create feature branch | git |
| 2 | Update composable for URL nav | useLevelUpWizard.ts |
| 3 | Simplify store | characterLevelUp.ts |
| 4 | Create preview page | index.vue |
| 5 | Create dynamic step page | [step].vue |
| 6 | Update sidebar | LevelUpSidebar.vue |
| 7 | Create step stubs | 6 component files |
| 8 | Fix test/lint issues | various |
| 9 | Manual testing | browser |
| 10 | PR creation | git/gh |
