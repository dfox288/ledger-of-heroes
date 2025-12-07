# Edit Level 1 Characters - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Allow users to edit level 1 characters by loading them into the character builder wizard.

**Architecture:** Unified edit page - "Create Character" becomes POST + redirect to `/characters/[id]/edit`. Single wizard page handles both new and existing characters.

**Tech Stack:** Nuxt 4, Pinia, TypeScript, Vitest, NuxtUI 4

**Issue:** #105

---

## Task 1: Add Store Actions for Loading Characters

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Test: `tests/stores/characterBuilder.test.ts`

### Step 1: Write failing test for loadCharacterForEditing

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
describe('loadCharacterForEditing', () => {
  it('loads character data into store', async () => {
    const mockCharacter = {
      id: 42,
      name: 'Gandalf',
      level: 1,
      ability_scores: { STR: 14, DEX: 12, CON: 15, INT: 18, WIS: 16, CHA: 10 },
      race: { id: 1, name: 'Human', slug: 'human' },
      class: { id: 2, name: 'Wizard', slug: 'wizard' },
      background: { id: 3, name: 'Sage', slug: 'sage' }
    }

    const mockRace = { id: 1, name: 'Human', slug: 'human', speed: 30 }
    const mockClass = { id: 2, name: 'Wizard', slug: 'wizard', spellcasting_ability: { id: 4, code: 'INT', name: 'Intelligence' } }
    const mockBackground = { id: 3, name: 'Sage', slug: 'sage' }

    mockApiFetch
      .mockResolvedValueOnce({ data: mockCharacter }) // GET /characters/42
      .mockResolvedValueOnce({ data: mockRace }) // GET /races/human
      .mockResolvedValueOnce({ data: mockClass }) // GET /classes/wizard
      .mockResolvedValueOnce({ data: mockBackground }) // GET /backgrounds/sage
      .mockResolvedValueOnce({ data: [] }) // GET /characters/42/spells

    const store = useCharacterBuilderStore()
    await store.loadCharacterForEditing(42)

    expect(store.characterId).toBe(42)
    expect(store.name).toBe('Gandalf')
    expect(store.abilityScores.strength).toBe(14)
    expect(store.abilityScores.intelligence).toBe(18)
    expect(store.raceId).toBe(1)
    expect(store.classId).toBe(2)
    expect(store.backgroundId).toBe(3)
    expect(store.selectedRace).toEqual(mockRace)
    expect(store.selectedClass).toEqual(mockClass)
  })

  it('throws error for characters above level 1', async () => {
    mockApiFetch.mockResolvedValueOnce({
      data: { id: 42, name: 'High Level', level: 5 }
    })

    const store = useCharacterBuilderStore()
    await expect(store.loadCharacterForEditing(42)).rejects.toThrow('Only level 1 characters can be edited')
  })

  it('handles subrace by setting both raceId and subraceId', async () => {
    const mockCharacter = {
      id: 42,
      name: 'Thorin',
      level: 1,
      ability_scores: { STR: 14, DEX: 10, CON: 16, INT: 10, WIS: 12, CHA: 8 },
      race: { id: 2, name: 'Hill Dwarf', slug: 'hill-dwarf' },
      class: null,
      background: null
    }

    const mockSubrace = {
      id: 2,
      name: 'Hill Dwarf',
      slug: 'hill-dwarf',
      parent_race: { id: 1, name: 'Dwarf', slug: 'dwarf' }
    }

    mockApiFetch
      .mockResolvedValueOnce({ data: mockCharacter })
      .mockResolvedValueOnce({ data: mockSubrace })

    const store = useCharacterBuilderStore()
    await store.loadCharacterForEditing(42)

    expect(store.raceId).toBe(1) // Parent race ID
    expect(store.subraceId).toBe(2) // Subrace ID
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose -t "loadCharacterForEditing"
```

Expected: FAIL with "store.loadCharacterForEditing is not a function"

### Step 3: Implement loadCharacterForEditing

Add to `app/stores/characterBuilder.ts` before the `return` statement:

```typescript
/**
 * Load an existing character for editing
 * Fetches character data and populates all wizard state
 */
async function loadCharacterForEditing(id: number): Promise<void> {
  isLoading.value = true
  error.value = null

  try {
    // 1. Fetch character data
    const response = await apiFetch<{ data: Character }>(`/characters/${id}`)
    const character = response.data

    // 2. Check level constraint
    if (character.level > 1) {
      throw new Error('Only level 1 characters can be edited')
    }

    // 3. Populate basic fields
    characterId.value = character.id
    name.value = character.name

    // 4. Map ability scores (API uses STR/DEX, store uses strength/dexterity)
    if (character.ability_scores) {
      abilityScores.value = {
        strength: character.ability_scores.STR ?? 10,
        dexterity: character.ability_scores.DEX ?? 10,
        constitution: character.ability_scores.CON ?? 10,
        intelligence: character.ability_scores.INT ?? 10,
        wisdom: character.ability_scores.WIS ?? 10,
        charisma: character.ability_scores.CHA ?? 10
      }
    }

    // 5. Fetch full reference data if relations exist
    if (character.race) {
      const raceResponse = await apiFetch<{ data: Race }>(`/races/${character.race.slug}`)
      selectedRace.value = raceResponse.data
      raceId.value = character.race.id

      // Handle subrace detection
      if (raceResponse.data.parent_race) {
        subraceId.value = character.race.id
        raceId.value = raceResponse.data.parent_race.id
      }
    }

    if (character.class) {
      const classResponse = await apiFetch<{ data: CharacterClass }>(`/classes/${character.class.slug}`)
      selectedClass.value = classResponse.data
      classId.value = character.class.id
    }

    if (character.background) {
      const bgResponse = await apiFetch<{ data: Background }>(`/backgrounds/${character.background.slug}`)
      selectedBackground.value = bgResponse.data
      backgroundId.value = character.background.id
    }

    // 6. Fetch spells if caster
    if (selectedClass.value?.spellcasting_ability) {
      await fetchSelectedSpells()
    }

    // 7. Determine starting step
    currentStep.value = determineStartingStep(character)
  } catch (err: unknown) {
    error.value = err instanceof Error ? err.message : 'Failed to load character'
    throw err
  } finally {
    isLoading.value = false
  }
}

/**
 * Determine which step to start on based on what's filled
 */
function determineStartingStep(character: Character): number {
  if (!character.name) return 1
  if (!character.race) return 2
  if (!character.class) return 3
  if (!character.ability_scores?.STR) return 4
  if (!character.background) return 5
  return 6 // Start at equipment for mostly-complete characters
}
```

Add to the return statement:
```typescript
return {
  // ... existing exports ...
  loadCharacterForEditing,
}
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose -t "loadCharacterForEditing"
```

Expected: PASS

### Step 5: Commit

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat(store): add loadCharacterForEditing action (#105)"
```

---

## Task 2: Add updateName Store Action

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Test: `tests/stores/characterBuilder.test.ts`

### Step 1: Write failing test for updateName

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
describe('updateName', () => {
  it('updates character name via API', async () => {
    mockApiFetch.mockResolvedValueOnce({ data: { id: 42, name: 'New Name' } })

    const store = useCharacterBuilderStore()
    store.characterId = 42
    store.name = 'Old Name'

    await store.updateName('New Name')

    expect(mockApiFetch).toHaveBeenCalledWith('/characters/42', {
      method: 'PATCH',
      body: { name: 'New Name' }
    })
    expect(store.name).toBe('New Name')
  })

  it('does nothing if no characterId', async () => {
    const store = useCharacterBuilderStore()
    store.characterId = null

    await store.updateName('New Name')

    expect(mockApiFetch).not.toHaveBeenCalled()
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose -t "updateName"
```

Expected: FAIL with "store.updateName is not a function"

### Step 3: Implement updateName

Add to `app/stores/characterBuilder.ts`:

```typescript
/**
 * Update character name (for edit mode)
 */
async function updateName(newName: string): Promise<void> {
  if (!characterId.value) return

  await apiFetch(`/characters/${characterId.value}`, {
    method: 'PATCH',
    body: { name: newName }
  })

  name.value = newName
}
```

Add to return statement: `updateName,`

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose -t "updateName"
```

Expected: PASS

### Step 5: Commit

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "feat(store): add updateName action (#105)"
```

---

## Task 3: Create Edit Page

**Files:**
- Create: `app/pages/characters/[id]/edit.vue`
- Test: `tests/pages/characters/edit.test.ts`

### Step 1: Write failing test for edit page

Create `tests/pages/characters/edit.test.ts`:

```typescript
// tests/pages/characters/edit.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { flushPromises } from '@vue/test-utils'
import { setActivePinia, createPinia } from 'pinia'

// Mock the store
const mockLoadCharacterForEditing = vi.fn()
const mockReset = vi.fn()

vi.mock('~/stores/characterBuilder', () => ({
  useCharacterBuilderStore: () => ({
    loadCharacterForEditing: mockLoadCharacterForEditing,
    reset: mockReset,
    currentStep: 1,
    isFirstStep: true,
    isLastStep: false,
    isCaster: false,
    isLoading: false,
    error: null,
    name: 'Test Character',
    characterId: 42
  })
}))

// Mock useRoute
vi.mock('vue-router', async () => {
  const actual = await vi.importActual('vue-router')
  return {
    ...actual,
    useRoute: () => ({
      params: { id: '42' }
    })
  }
})

describe('CharacterEditPage', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    mockLoadCharacterForEditing.mockReset()
    mockReset.mockReset()
  })

  it('calls loadCharacterForEditing on mount', async () => {
    mockLoadCharacterForEditing.mockResolvedValue(undefined)

    const CharacterEditPage = await import('~/pages/characters/[id]/edit.vue')
    await mountSuspended(CharacterEditPage.default)
    await flushPromises()

    expect(mockReset).toHaveBeenCalled()
    expect(mockLoadCharacterForEditing).toHaveBeenCalledWith(42)
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm run test -- tests/pages/characters/edit.test.ts --reporter=verbose
```

Expected: FAIL (page doesn't exist)

### Step 3: Create edit page

Create `app/pages/characters/[id]/edit.vue`:

```vue
<!-- app/pages/characters/[id]/edit.vue -->
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

const route = useRoute()
const characterId = computed(() => Number(route.params.id))

// Page metadata
useSeoMeta({
  title: 'Edit Character',
  description: 'Continue building your D&D 5e character'
})

// Store
const store = useCharacterBuilderStore()
const { currentStep, isFirstStep, isLastStep, isCaster, isLoading, error, name } = storeToRefs(store)

// Load character on mount
onMounted(async () => {
  store.reset()

  try {
    await store.loadCharacterForEditing(characterId.value)
  } catch {
    await navigateTo(`/characters/${characterId.value}`)
  }
})

// Step definitions
const steps = computed(() => {
  const baseSteps = [
    { id: 1, name: 'name', label: 'Name', icon: 'i-heroicons-user' },
    { id: 2, name: 'race', label: 'Race', icon: 'i-heroicons-globe-alt' },
    { id: 3, name: 'class', label: 'Class', icon: 'i-heroicons-shield-check' },
    { id: 4, name: 'abilities', label: 'Abilities', icon: 'i-heroicons-chart-bar' },
    { id: 5, name: 'background', label: 'Background', icon: 'i-heroicons-book-open' },
    { id: 6, name: 'equipment', label: 'Equipment', icon: 'i-heroicons-briefcase' }
  ]

  if (isCaster.value) {
    baseSteps.push({ id: 7, name: 'spells', label: 'Spells', icon: 'i-heroicons-sparkles' })
  }

  baseSteps.push({
    id: isCaster.value ? 8 : 7,
    name: 'review',
    label: 'Review',
    icon: 'i-heroicons-check-circle'
  })

  return baseSteps
})
</script>

<template>
  <div class="container mx-auto px-4 py-8 max-w-4xl">
    <!-- Loading State -->
    <div v-if="isLoading" class="flex justify-center items-center py-12">
      <UIcon name="i-heroicons-arrow-path" class="w-8 h-8 animate-spin text-primary" />
      <span class="ml-3 text-gray-600 dark:text-gray-400">Loading character...</span>
    </div>

    <!-- Error State -->
    <UAlert v-else-if="error" color="error" :title="error" class="mb-6" />

    <!-- Wizard Content -->
    <template v-else>
      <!-- Page Header -->
      <div class="text-center mb-8">
        <h1 class="text-3xl font-bold text-gray-900 dark:text-white">
          Edit Character
        </h1>
        <p class="mt-2 text-gray-600 dark:text-gray-400">
          {{ name || 'Continue building your hero' }}
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
        <CharacterBuilderStepName v-if="currentStep === 1" />
        <CharacterBuilderStepRace v-else-if="currentStep === 2" />
        <CharacterBuilderStepClass v-else-if="currentStep === 3" />
        <CharacterBuilderStepAbilities v-else-if="currentStep === 4" />
        <CharacterBuilderStepBackground v-else-if="currentStep === 5" />
        <CharacterBuilderStepEquipment v-else-if="currentStep === 6" />

        <template v-else-if="currentStep === 7">
          <CharacterBuilderStepSpells v-if="isCaster" />
          <CharacterBuilderStepReview v-else />
        </template>

        <CharacterBuilderStepReview v-else-if="currentStep === 8" />
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
    </template>
  </div>
</template>
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm run test -- tests/pages/characters/edit.test.ts --reporter=verbose
```

Expected: PASS

### Step 5: Commit

```bash
git add app/pages/characters/[id]/edit.vue tests/pages/characters/edit.test.ts
git commit -m "feat(pages): add character edit page (#105)"
```

---

## Task 4: Modify Create Page to Redirect

**Files:**
- Modify: `app/pages/characters/create.vue`
- Modify: `tests/pages/characters/create.test.ts`

### Step 1: Update create page tests

Replace `tests/pages/characters/create.test.ts` content:

```typescript
// tests/pages/characters/create.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { flushPromises } from '@vue/test-utils'
import { setActivePinia, createPinia } from 'pinia'

// Mock navigateTo
const mockNavigateTo = vi.fn()
vi.stubGlobal('navigateTo', mockNavigateTo)

// Mock useApi
const mockApiFetch = vi.fn()
vi.mock('~/composables/useApi', () => ({
  useApi: () => ({
    apiFetch: mockApiFetch
  })
}))

describe('CharacterCreatePage', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    mockApiFetch.mockReset()
    mockNavigateTo.mockReset()
  })

  it('creates character and redirects to edit page', async () => {
    mockApiFetch.mockResolvedValueOnce({ data: { id: 42 } })

    const CharacterCreatePage = await import('~/pages/characters/create.vue')
    await mountSuspended(CharacterCreatePage.default)
    await flushPromises()

    expect(mockApiFetch).toHaveBeenCalledWith('/characters', {
      method: 'POST',
      body: { name: '' }
    })
    expect(mockNavigateTo).toHaveBeenCalledWith('/characters/42/edit')
  })

  it('redirects to character list on error', async () => {
    mockApiFetch.mockRejectedValueOnce(new Error('Failed'))

    const CharacterCreatePage = await import('~/pages/characters/create.vue')
    await mountSuspended(CharacterCreatePage.default)
    await flushPromises()

    expect(mockNavigateTo).toHaveBeenCalledWith('/characters')
  })
})
```

### Step 2: Run test to verify it fails

```bash
docker compose exec nuxt npm run test -- tests/pages/characters/create.test.ts --reporter=verbose
```

Expected: FAIL (old behavior)

### Step 3: Simplify create page

Replace `app/pages/characters/create.vue`:

```vue
<!-- app/pages/characters/create.vue -->
<script setup lang="ts">
const { apiFetch } = useApi()

useSeoMeta({
  title: 'Create Character',
  description: 'Start building your D&D 5e character'
})

// Create empty character and redirect to edit
onMounted(async () => {
  try {
    const response = await apiFetch<{ data: { id: number } }>('/characters', {
      method: 'POST',
      body: { name: '' }
    })

    await navigateTo(`/characters/${response.data.id}/edit`)
  } catch {
    await navigateTo('/characters')
  }
})
</script>

<template>
  <div class="flex justify-center items-center min-h-[50vh]">
    <UIcon name="i-heroicons-arrow-path" class="w-8 h-8 animate-spin text-primary" />
    <span class="ml-3 text-gray-600 dark:text-gray-400">Creating character...</span>
  </div>
</template>
```

### Step 4: Run test to verify it passes

```bash
docker compose exec nuxt npm run test -- tests/pages/characters/create.test.ts --reporter=verbose
```

Expected: PASS

### Step 5: Commit

```bash
git add app/pages/characters/create.vue tests/pages/characters/create.test.ts
git commit -m "refactor(pages): simplify create page to POST + redirect (#105)"
```

---

## Task 5: Update StepName for Edit Mode

**Files:**
- Modify: `app/components/character/builder/StepName.vue`
- Test: `tests/components/character/builder/StepName.test.ts`

### Step 1: Read current StepName implementation

```bash
cat app/components/character/builder/StepName.vue
```

### Step 2: Write test for edit mode behavior

Create or update `tests/components/character/builder/StepName.test.ts`:

```typescript
// tests/components/character/builder/StepName.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepName from '~/components/character/builder/StepName.vue'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

// Mock useApi
const mockApiFetch = vi.fn()
vi.mock('~/composables/useApi', () => ({
  useApi: () => ({
    apiFetch: mockApiFetch
  })
}))

describe('StepName', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
    mockApiFetch.mockReset()
  })

  it('calls updateName when editing existing character', async () => {
    mockApiFetch.mockResolvedValue({ data: { id: 42, name: 'Updated' } })

    const store = useCharacterBuilderStore()
    store.characterId = 42 // Existing character
    store.name = 'Original Name'

    const wrapper = await mountSuspended(StepName)

    // Find input and update value
    const input = wrapper.find('input')
    await input.setValue('Updated Name')

    // Submit form
    await wrapper.find('form').trigger('submit')

    // Should call PATCH, not POST
    expect(mockApiFetch).toHaveBeenCalledWith('/characters/42', {
      method: 'PATCH',
      body: { name: 'Updated Name' }
    })
  })

  it('calls createDraft when creating new character', async () => {
    mockApiFetch.mockResolvedValue({ data: { id: 99, name: 'New Character' } })

    const store = useCharacterBuilderStore()
    store.characterId = null // No existing character

    const wrapper = await mountSuspended(StepName)

    const input = wrapper.find('input')
    await input.setValue('New Character')

    await wrapper.find('form').trigger('submit')

    // Should call POST
    expect(mockApiFetch).toHaveBeenCalledWith('/characters', {
      method: 'POST',
      body: { name: 'New Character' }
    })
  })
})
```

### Step 3: Run test to verify behavior

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/StepName.test.ts --reporter=verbose
```

### Step 4: Update StepName component

Modify the submit handler in `app/components/character/builder/StepName.vue` to check for edit mode:

```typescript
async function handleSubmit() {
  if (!characterName.value.trim()) return

  isSubmitting.value = true
  try {
    if (store.characterId) {
      // Edit mode - update existing character
      await store.updateName(characterName.value)
    } else {
      // Create mode - this shouldn't happen with new flow,
      // but keep for backwards compatibility
      await store.createDraft(characterName.value)
    }
    store.nextStep()
  } catch (err) {
    console.error('Failed to save name:', err)
  } finally {
    isSubmitting.value = false
  }
}
```

### Step 5: Run test to verify it passes

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/StepName.test.ts --reporter=verbose
```

Expected: PASS

### Step 6: Commit

```bash
git add app/components/character/builder/StepName.vue tests/components/character/builder/StepName.test.ts
git commit -m "feat(StepName): support edit mode with updateName (#105)"
```

---

## Task 6: Add Edit Button to Character Card

**Files:**
- Modify: `app/components/character/CharacterCard.vue`
- Test: `tests/components/character/CharacterCard.test.ts`

### Step 1: Read current CharacterCard implementation

```bash
cat app/components/character/CharacterCard.vue
```

### Step 2: Add Edit button for level 1 characters

Add button in the card actions area:

```vue
<!-- In the actions/buttons section -->
<UButton
  v-if="character.level === 1"
  :to="`/characters/${character.id}/edit`"
  color="primary"
  size="sm"
>
  {{ character.is_complete ? 'Edit' : 'Continue' }}
</UButton>
```

### Step 3: Write test for edit button

Add to character card tests:

```typescript
it('shows Edit button for level 1 complete characters', async () => {
  const wrapper = await mountSuspended(CharacterCard, {
    props: {
      character: { id: 1, name: 'Test', level: 1, is_complete: true }
    }
  })

  const editButton = wrapper.find('a[href="/characters/1/edit"]')
  expect(editButton.exists()).toBe(true)
  expect(editButton.text()).toContain('Edit')
})

it('shows Continue button for level 1 incomplete characters', async () => {
  const wrapper = await mountSuspended(CharacterCard, {
    props: {
      character: { id: 1, name: 'Test', level: 1, is_complete: false }
    }
  })

  const editButton = wrapper.find('a[href="/characters/1/edit"]')
  expect(editButton.exists()).toBe(true)
  expect(editButton.text()).toContain('Continue')
})

it('hides Edit button for characters above level 1', async () => {
  const wrapper = await mountSuspended(CharacterCard, {
    props: {
      character: { id: 1, name: 'Test', level: 5, is_complete: true }
    }
  })

  const editButton = wrapper.find('a[href="/characters/1/edit"]')
  expect(editButton.exists()).toBe(false)
})
```

### Step 4: Run tests

```bash
docker compose exec nuxt npm run test -- tests/components/character/CharacterCard.test.ts --reporter=verbose
```

### Step 5: Commit

```bash
git add app/components/character/CharacterCard.vue tests/components/character/CharacterCard.test.ts
git commit -m "feat(CharacterCard): add Edit/Continue button for level 1 (#105)"
```

---

## Task 7: Add Class Change Warning in StepClass

**Files:**
- Modify: `app/components/character/builder/StepClass.vue`

### Step 1: Read current StepClass implementation

```bash
cat app/components/character/builder/StepClass.vue
```

### Step 2: Add confirmation dialog for class change

Add modal state and handler:

```typescript
const showClassChangeWarning = ref(false)
const pendingClass = ref<CharacterClass | null>(null)

async function handleClassSelect(cls: CharacterClass) {
  // If editing and class is changing, show warning
  if (store.classId && store.classId !== cls.id) {
    pendingClass.value = cls
    showClassChangeWarning.value = true
    return
  }

  await selectClass(cls)
}

async function confirmClassChange() {
  if (!pendingClass.value) return

  // Clear equipment and spell selections
  store.equipmentChoices.clear()
  store.equipmentItemSelections.clear()
  store.selectedSpells = []

  await selectClass(pendingClass.value)
  showClassChangeWarning.value = false
  pendingClass.value = null
}

function cancelClassChange() {
  showClassChangeWarning.value = false
  pendingClass.value = null
}

async function selectClass(cls: CharacterClass) {
  await store.selectClass(cls)
  store.nextStep()
}
```

Add modal template:

```vue
<!-- Class change warning modal -->
<UModal v-model:open="showClassChangeWarning">
  <template #header>
    <h3 class="text-lg font-semibold">Change Class?</h3>
  </template>

  <p class="text-gray-600 dark:text-gray-400">
    Changing your class will reset your equipment and spell selections.
    You'll need to choose them again.
  </p>

  <template #footer>
    <div class="flex justify-end gap-3">
      <UButton variant="outline" @click="cancelClassChange">
        Cancel
      </UButton>
      <UButton color="primary" @click="confirmClassChange">
        Change Class
      </UButton>
    </div>
  </template>
</UModal>
```

### Step 3: Commit

```bash
git add app/components/character/builder/StepClass.vue
git commit -m "feat(StepClass): add class change warning dialog (#105)"
```

---

## Task 8: Final Verification and Cleanup

### Step 1: Run full test suite

```bash
docker compose exec nuxt npm run test
```

Expected: All tests pass

### Step 2: Run typecheck

```bash
docker compose exec nuxt npm run typecheck
```

Expected: No errors

### Step 3: Run linter

```bash
docker compose exec nuxt npm run lint:fix
```

### Step 4: Manual browser testing

1. Go to `/characters`
2. Click "Create Character" - should redirect to `/characters/[id]/edit`
3. Fill out wizard steps
4. Navigate away, come back to `/characters`
5. Click "Continue" on incomplete character - should resume
6. Click "Edit" on complete level 1 character - should load in wizard
7. Try changing class - should show warning dialog

### Step 5: Final commit and push

```bash
git add -A
git commit -m "chore: final cleanup for edit level 1 characters (#105)"
git push -u origin feature/issue-105-edit-level1-characters
```

### Step 6: Create PR

```bash
gh pr create --title "feat: Allow editing level 1 characters (#105)" --body "$(cat <<'EOF'
## Summary
- Added ability to edit level 1 characters in the wizard
- "Create Character" now creates empty character and redirects to edit page
- Single unified edit page for both new and existing characters

## Changes
- Added `loadCharacterForEditing()` store action
- New `/characters/[id]/edit` page
- Simplified `/characters/create` to POST + redirect
- Added Edit/Continue button to character cards (level 1 only)
- Added class change warning dialog

## Test Plan
- [x] Store tests for loadCharacterForEditing
- [x] Page tests for edit and create pages
- [x] Component tests for StepName edit mode
- [x] Manual browser testing

Closes #105

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

---

## Summary

| Task | Description | Est. Time |
|------|-------------|-----------|
| 1 | Add loadCharacterForEditing store action | 10 min |
| 2 | Add updateName store action | 5 min |
| 3 | Create edit page | 10 min |
| 4 | Modify create page to redirect | 5 min |
| 5 | Update StepName for edit mode | 5 min |
| 6 | Add edit button to character card | 5 min |
| 7 | Add class change warning | 5 min |
| 8 | Final verification | 10 min |

**Total: ~55 minutes**
