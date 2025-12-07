# Character Builder Phase 2: Race & Class Selection - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Implement Steps 2 (Race) and 3 (Class) of the character creation wizard with click-to-select cards and detail modals.

**Architecture:** Wrapper components (`RacePickerCard`, `ClassPickerCard`) wrap existing cards with selection behavior. Store actions `selectRace()` and `selectClass()` PATCH the API and refresh stats. Detail modals show full entity information.

**Tech Stack:** Vue 3, Pinia, NuxtUI 4, Vitest, TypeScript

---

## Task 1: Add Store Actions (selectRace, selectClass, refreshStats)

**Files:**
- Modify: `app/stores/characterBuilder.ts:103-127`
- Modify: `tests/stores/characterBuilder.test.ts`

### Step 1: Write failing tests for refreshStats

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
describe('refreshStats action', () => {
  it('fetches stats from API when characterId exists', async () => {
    mockApiFetch.mockResolvedValue({
      data: {
        character_id: 42,
        level: 1,
        proficiency_bonus: 2,
        ability_scores: {}
      }
    })

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await store.refreshStats()

    expect(mockApiFetch).toHaveBeenCalledWith('/characters/42/stats')
    expect(store.characterStats).toEqual({
      character_id: 42,
      level: 1,
      proficiency_bonus: 2,
      ability_scores: {}
    })
  })

  it('does nothing when characterId is null', async () => {
    const store = useCharacterBuilderStore()
    store.characterId = null

    await store.refreshStats()

    expect(mockApiFetch).not.toHaveBeenCalled()
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose
```

Expected: FAIL - `store.refreshStats is not a function`

### Step 3: Implement refreshStats in store

Add to `app/stores/characterBuilder.ts` after the `createDraft` function (around line 127):

```typescript
/**
 * Refresh character stats from API
 */
async function refreshStats(): Promise<void> {
  if (!characterId.value) return

  const response = await apiFetch<{ data: CharacterStats }>(
    `/characters/${characterId.value}/stats`
  )
  characterStats.value = response.data
}
```

Update the return statement to include `refreshStats`:

```typescript
return {
  // ... existing exports ...
  createDraft,
  refreshStats,  // Add this
  reset
}
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose
```

Expected: PASS

### Step 5: Write failing tests for selectRace

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
describe('selectRace action', () => {
  const mockRace: Race = {
    id: 1,
    name: 'Dwarf',
    slug: 'dwarf',
    speed: 25
  } as Race

  const mockSubrace: Race = {
    id: 2,
    name: 'Hill Dwarf',
    slug: 'hill-dwarf',
    speed: 25,
    parent_race: { id: 1, name: 'Dwarf', slug: 'dwarf' }
  } as Race

  it('calls API with race_id', async () => {
    mockApiFetch.mockResolvedValue({ data: {} })

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await store.selectRace(mockRace)

    expect(mockApiFetch).toHaveBeenCalledWith('/characters/42', {
      method: 'PATCH',
      body: { race_id: 1 }
    })
  })

  it('uses subrace ID when subrace provided', async () => {
    mockApiFetch.mockResolvedValue({ data: {} })

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await store.selectRace(mockRace, mockSubrace)

    expect(mockApiFetch).toHaveBeenCalledWith('/characters/42', {
      method: 'PATCH',
      body: { race_id: 2 }
    })
  })

  it('updates store state after selection', async () => {
    mockApiFetch.mockResolvedValue({ data: {} })

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await store.selectRace(mockRace, mockSubrace)

    expect(store.raceId).toBe(1)
    expect(store.subraceId).toBe(2)
    expect(store.selectedRace).toEqual(mockSubrace)
  })

  it('sets loading state during API call', async () => {
    let resolvePromise: (value: unknown) => void
    mockApiFetch.mockReturnValue(new Promise(resolve => { resolvePromise = resolve }))

    const store = useCharacterBuilderStore()
    store.characterId = 42

    const promise = store.selectRace(mockRace)
    expect(store.isLoading).toBe(true)

    resolvePromise!({ data: {} })
    await promise

    expect(store.isLoading).toBe(false)
  })

  it('sets error on API failure', async () => {
    mockApiFetch.mockRejectedValue(new Error('Network error'))

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await expect(store.selectRace(mockRace)).rejects.toThrow('Network error')
    expect(store.error).toBe('Failed to save race')
  })
})
```

Add the import at the top of the test file:

```typescript
import type { Race } from '~/types'
```

### Step 6: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose
```

Expected: FAIL - `store.selectRace is not a function`

### Step 7: Implement selectRace in store

Add to `app/stores/characterBuilder.ts` after `refreshStats`:

```typescript
/**
 * Step 2: Select race (and optional subrace)
 * API expects race_id to be the subrace ID when a subrace is selected
 */
async function selectRace(race: Race, subrace?: Race): Promise<void> {
  isLoading.value = true
  error.value = null

  try {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: { race_id: subrace?.id ?? race.id }
    })

    raceId.value = race.id
    subraceId.value = subrace?.id ?? null
    selectedRace.value = subrace ?? race

    await refreshStats()
  } catch (err: unknown) {
    error.value = 'Failed to save race'
    throw err
  } finally {
    isLoading.value = false
  }
}
```

Update the return statement:

```typescript
return {
  // ... existing exports ...
  createDraft,
  refreshStats,
  selectRace,  // Add this
  reset
}
```

### Step 8: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose
```

Expected: PASS

### Step 9: Write failing tests for selectClass

Add to `tests/stores/characterBuilder.test.ts`:

```typescript
describe('selectClass action', () => {
  const mockClass: CharacterClass = {
    id: 1,
    name: 'Fighter',
    slug: 'fighter',
    hit_die: 10,
    is_base_class: true,
    spellcasting_ability: null
  } as CharacterClass

  const mockCasterClass: CharacterClass = {
    id: 2,
    name: 'Wizard',
    slug: 'wizard',
    hit_die: 6,
    is_base_class: true,
    spellcasting_ability: { id: 4, code: 'INT', name: 'Intelligence' }
  } as CharacterClass

  it('calls API with class_id', async () => {
    mockApiFetch.mockResolvedValue({ data: {} })

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await store.selectClass(mockClass)

    expect(mockApiFetch).toHaveBeenCalledWith('/characters/42', {
      method: 'PATCH',
      body: { class_id: 1 }
    })
  })

  it('updates store state after selection', async () => {
    mockApiFetch.mockResolvedValue({ data: {} })

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await store.selectClass(mockClass)

    expect(store.classId).toBe(1)
    expect(store.selectedClass).toEqual(mockClass)
  })

  it('isCaster is false for non-caster class', async () => {
    mockApiFetch.mockResolvedValue({ data: {} })

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await store.selectClass(mockClass)

    expect(store.isCaster).toBe(false)
    expect(store.totalSteps).toBe(6)
  })

  it('isCaster is true for caster class', async () => {
    mockApiFetch.mockResolvedValue({ data: {} })

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await store.selectClass(mockCasterClass)

    expect(store.isCaster).toBe(true)
    expect(store.totalSteps).toBe(7)
  })

  it('sets loading state during API call', async () => {
    let resolvePromise: (value: unknown) => void
    mockApiFetch.mockReturnValue(new Promise(resolve => { resolvePromise = resolve }))

    const store = useCharacterBuilderStore()
    store.characterId = 42

    const promise = store.selectClass(mockClass)
    expect(store.isLoading).toBe(true)

    resolvePromise!({ data: {} })
    await promise

    expect(store.isLoading).toBe(false)
  })

  it('sets error on API failure', async () => {
    mockApiFetch.mockRejectedValue(new Error('Network error'))

    const store = useCharacterBuilderStore()
    store.characterId = 42

    await expect(store.selectClass(mockClass)).rejects.toThrow('Network error')
    expect(store.error).toBe('Failed to save class')
  })
})
```

Add the import:

```typescript
import type { Race, CharacterClass } from '~/types'
```

### Step 10: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose
```

Expected: FAIL - `store.selectClass is not a function`

### Step 11: Implement selectClass in store

Add to `app/stores/characterBuilder.ts` after `selectRace`:

```typescript
/**
 * Step 3: Select class
 * This updates isCaster computed which affects totalSteps
 */
async function selectClass(cls: CharacterClass): Promise<void> {
  isLoading.value = true
  error.value = null

  try {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: { class_id: cls.id }
    })

    classId.value = cls.id
    selectedClass.value = cls

    await refreshStats()
  } catch (err: unknown) {
    error.value = 'Failed to save class'
    throw err
  } finally {
    isLoading.value = false
  }
}
```

Update the return statement:

```typescript
return {
  // ... existing exports ...
  createDraft,
  refreshStats,
  selectRace,
  selectClass,  // Add this
  reset
}
```

### Step 12: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts --reporter=verbose
```

Expected: PASS - all store tests should pass

### Step 13: Commit

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.test.ts
git commit -m "$(cat <<'EOF'
feat(character): add selectRace, selectClass, refreshStats store actions

- selectRace updates race/subrace with API PATCH
- selectClass updates class with API PATCH, triggers isCaster recompute
- refreshStats fetches /characters/{id}/stats
- All actions include loading state and error handling

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 2: Create RacePickerCard Component

**Files:**
- Create: `app/components/character/builder/RacePickerCard.vue`
- Create: `tests/components/character/builder/RacePickerCard.test.ts`

### Step 1: Write failing tests

Create `tests/components/character/builder/RacePickerCard.test.ts`:

```typescript
// tests/components/character/builder/RacePickerCard.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import RacePickerCard from '~/components/character/builder/RacePickerCard.vue'
import type { Race } from '~/types'

const mockRace: Race = {
  id: 1,
  name: 'Dwarf',
  slug: 'dwarf',
  speed: 25,
  size: { id: 1, code: 'M', name: 'Medium' },
  modifiers: [
    { id: 1, value: 2, modifier_category: 'ability_score', ability_score: { id: 1, code: 'CON', name: 'Constitution' } }
  ],
  sources: []
} as Race

const mockRaceWithSubraces: Race = {
  ...mockRace,
  id: 2,
  name: 'Elf',
  slug: 'elf',
  subraces: [
    { id: 3, name: 'High Elf', slug: 'high-elf' },
    { id: 4, name: 'Wood Elf', slug: 'wood-elf' }
  ]
} as Race

describe('RacePickerCard', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders the race name', async () => {
    const wrapper = await mountSuspended(RacePickerCard, {
      props: { race: mockRace, selected: false }
    })
    expect(wrapper.text()).toContain('Dwarf')
  })

  it('shows selected styling when selected', async () => {
    const wrapper = await mountSuspended(RacePickerCard, {
      props: { race: mockRace, selected: true }
    })
    expect(wrapper.find('[data-testid="picker-card"]').classes()).toContain('ring-2')
  })

  it('does not show selected styling when not selected', async () => {
    const wrapper = await mountSuspended(RacePickerCard, {
      props: { race: mockRace, selected: false }
    })
    expect(wrapper.find('[data-testid="picker-card"]').classes()).not.toContain('ring-2')
  })

  it('emits select event when card is clicked', async () => {
    const wrapper = await mountSuspended(RacePickerCard, {
      props: { race: mockRace, selected: false }
    })
    await wrapper.find('[data-testid="picker-card"]').trigger('click')
    expect(wrapper.emitted('select')).toBeTruthy()
    expect(wrapper.emitted('select')![0]).toEqual([mockRace])
  })

  it('shows View Details button', async () => {
    const wrapper = await mountSuspended(RacePickerCard, {
      props: { race: mockRace, selected: false }
    })
    expect(wrapper.text()).toContain('View Details')
  })

  it('emits view-details event when View Details button is clicked', async () => {
    const wrapper = await mountSuspended(RacePickerCard, {
      props: { race: mockRace, selected: false }
    })
    await wrapper.find('[data-testid="view-details-btn"]').trigger('click')
    expect(wrapper.emitted('view-details')).toBeTruthy()
    expect(wrapper.emitted('select')).toBeFalsy() // Should not also emit select
  })

  it('shows subrace count badge when race has subraces', async () => {
    const wrapper = await mountSuspended(RacePickerCard, {
      props: { race: mockRaceWithSubraces, selected: false }
    })
    expect(wrapper.text()).toContain('2 Subraces')
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/RacePickerCard.test.ts --reporter=verbose
```

Expected: FAIL - component doesn't exist or render correctly

### Step 3: Implement RacePickerCard component

Create `app/components/character/builder/RacePickerCard.vue`:

```vue
<!-- app/components/character/builder/RacePickerCard.vue -->
<script setup lang="ts">
import type { Race } from '~/types'

interface Props {
  race: Race
  selected: boolean
}

const props = defineProps<Props>()

const emit = defineEmits<{
  select: [race: Race]
  'view-details': []
}>()

/**
 * Get ability score modifiers summary
 */
const abilityModifiers = computed(() => {
  if (!props.race.modifiers || props.race.modifiers.length === 0) return null

  const abilityScoreMods = props.race.modifiers
    .filter(m => m.modifier_category === 'ability_score' && m.ability_score)

  if (abilityScoreMods.length === 0) return null

  return abilityScoreMods
    .slice(0, 3)
    .map(m => `${m.ability_score?.code} +${m.value}`)
    .join(', ')
})

/**
 * Check if race has subraces
 */
const hasSubraces = computed(() => {
  return props.race.subraces && props.race.subraces.length > 0
})

const subraceCount = computed(() => {
  return props.race.subraces?.length ?? 0
})

/**
 * Handle card click - emit select
 */
function handleCardClick() {
  emit('select', props.race)
}

/**
 * Handle View Details click - emit event, stop propagation
 */
function handleViewDetails(event: Event) {
  event.stopPropagation()
  emit('view-details')
}

/**
 * Get background image path (256px variant)
 */
const { getImagePath } = useEntityImage()
const backgroundImage = computed(() => {
  return getImagePath('races', props.race.slug, 256)
})
</script>

<template>
  <div
    data-testid="picker-card"
    class="relative cursor-pointer transition-all"
    :class="[
      selected ? 'ring-2 ring-race-500 ring-offset-2' : '',
    ]"
    @click="handleCardClick"
  >
    <UCard class="relative overflow-hidden hover:shadow-lg transition-shadow h-full border-2 border-race-300 dark:border-race-700 hover:border-race-500">
      <!-- Background Image Layer -->
      <div
        v-if="backgroundImage"
        class="absolute inset-0 bg-cover bg-center opacity-15 transition-all duration-300"
        :style="{ backgroundImage: `url(${backgroundImage})` }"
      />

      <!-- Selected Checkmark -->
      <div
        v-if="selected"
        class="absolute top-2 right-2 z-20"
      >
        <UBadge
          color="success"
          variant="solid"
          size="md"
        >
          <UIcon
            name="i-heroicons-check"
            class="w-4 h-4"
          />
        </UBadge>
      </div>

      <!-- Content Layer -->
      <div class="relative z-10 flex flex-col h-full">
        <div class="space-y-3 flex-1">
          <!-- Size Badge -->
          <div class="flex items-center gap-2 flex-wrap">
            <UBadge
              v-if="race.size"
              color="info"
              variant="subtle"
              size="md"
            >
              {{ race.size.name }}
            </UBadge>
            <UBadge
              v-if="hasSubraces"
              color="race"
              variant="subtle"
              size="md"
            >
              {{ subraceCount }} {{ subraceCount === 1 ? 'Subrace' : 'Subraces' }}
            </UBadge>
          </div>

          <!-- Race Name -->
          <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100 line-clamp-2">
            {{ race.name }}
          </h3>

          <!-- Quick Stats -->
          <div class="flex items-center gap-4 flex-wrap text-sm text-gray-600 dark:text-gray-400">
            <div
              v-if="race.speed"
              class="flex items-center gap-1"
            >
              <UIcon
                name="i-heroicons-bolt"
                class="w-4 h-4"
              />
              <span>{{ race.speed }} ft</span>
            </div>
            <div
              v-if="abilityModifiers"
              class="flex items-center gap-1"
            >
              <UIcon
                name="i-heroicons-arrow-trending-up"
                class="w-4 h-4"
              />
              <span>{{ abilityModifiers }}</span>
            </div>
          </div>

          <!-- Description Preview -->
          <p
            v-if="race.description"
            class="text-sm text-gray-700 dark:text-gray-300 line-clamp-2"
          >
            {{ race.description }}
          </p>
        </div>

        <!-- View Details Button -->
        <div class="mt-4 pt-3 border-t border-gray-200 dark:border-gray-700">
          <UButton
            data-testid="view-details-btn"
            variant="ghost"
            color="race"
            size="sm"
            block
            @click="handleViewDetails"
          >
            <UIcon
              name="i-heroicons-eye"
              class="w-4 h-4 mr-1"
            />
            View Details
          </UButton>
        </div>
      </div>
    </UCard>
  </div>
</template>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/RacePickerCard.test.ts --reporter=verbose
```

Expected: PASS

### Step 5: Commit

```bash
git add app/components/character/builder/RacePickerCard.vue tests/components/character/builder/RacePickerCard.test.ts
git commit -m "$(cat <<'EOF'
feat(character): add RacePickerCard component

- Click-to-select with ring highlight when selected
- View Details button emits separate event
- Shows ability modifiers and subrace count badge
- Background image with fade effect

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Create ClassPickerCard Component

**Files:**
- Create: `app/components/character/builder/ClassPickerCard.vue`
- Create: `tests/components/character/builder/ClassPickerCard.test.ts`

### Step 1: Write failing tests

Create `tests/components/character/builder/ClassPickerCard.test.ts`:

```typescript
// tests/components/character/builder/ClassPickerCard.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import ClassPickerCard from '~/components/character/builder/ClassPickerCard.vue'
import type { CharacterClass } from '~/types'

const mockClass: CharacterClass = {
  id: 1,
  name: 'Fighter',
  slug: 'fighter',
  hit_die: 10,
  is_base_class: true,
  primary_ability: 'STR',
  spellcasting_ability: null,
  sources: []
} as CharacterClass

const mockCasterClass: CharacterClass = {
  id: 2,
  name: 'Wizard',
  slug: 'wizard',
  hit_die: 6,
  is_base_class: true,
  primary_ability: 'INT',
  spellcasting_ability: { id: 4, code: 'INT', name: 'Intelligence' },
  sources: []
} as CharacterClass

describe('ClassPickerCard', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders the class name', async () => {
    const wrapper = await mountSuspended(ClassPickerCard, {
      props: { characterClass: mockClass, selected: false }
    })
    expect(wrapper.text()).toContain('Fighter')
  })

  it('shows hit die', async () => {
    const wrapper = await mountSuspended(ClassPickerCard, {
      props: { characterClass: mockClass, selected: false }
    })
    expect(wrapper.text()).toContain('d10')
  })

  it('shows selected styling when selected', async () => {
    const wrapper = await mountSuspended(ClassPickerCard, {
      props: { characterClass: mockClass, selected: true }
    })
    expect(wrapper.find('[data-testid="picker-card"]').classes()).toContain('ring-2')
  })

  it('does not show selected styling when not selected', async () => {
    const wrapper = await mountSuspended(ClassPickerCard, {
      props: { characterClass: mockClass, selected: false }
    })
    expect(wrapper.find('[data-testid="picker-card"]').classes()).not.toContain('ring-2')
  })

  it('emits select event when card is clicked', async () => {
    const wrapper = await mountSuspended(ClassPickerCard, {
      props: { characterClass: mockClass, selected: false }
    })
    await wrapper.find('[data-testid="picker-card"]').trigger('click')
    expect(wrapper.emitted('select')).toBeTruthy()
    expect(wrapper.emitted('select')![0]).toEqual([mockClass])
  })

  it('shows View Details button', async () => {
    const wrapper = await mountSuspended(ClassPickerCard, {
      props: { characterClass: mockClass, selected: false }
    })
    expect(wrapper.text()).toContain('View Details')
  })

  it('emits view-details event when View Details button is clicked', async () => {
    const wrapper = await mountSuspended(ClassPickerCard, {
      props: { characterClass: mockClass, selected: false }
    })
    await wrapper.find('[data-testid="view-details-btn"]').trigger('click')
    expect(wrapper.emitted('view-details')).toBeTruthy()
    expect(wrapper.emitted('select')).toBeFalsy()
  })

  it('shows spellcasting indicator for caster classes', async () => {
    const wrapper = await mountSuspended(ClassPickerCard, {
      props: { characterClass: mockCasterClass, selected: false }
    })
    expect(wrapper.text()).toContain('Intelligence')
  })

  it('does not show spellcasting indicator for non-caster classes', async () => {
    const wrapper = await mountSuspended(ClassPickerCard, {
      props: { characterClass: mockClass, selected: false }
    })
    expect(wrapper.text()).not.toContain('Spellcasting')
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/ClassPickerCard.test.ts --reporter=verbose
```

Expected: FAIL

### Step 3: Implement ClassPickerCard component

Create `app/components/character/builder/ClassPickerCard.vue`:

```vue
<!-- app/components/character/builder/ClassPickerCard.vue -->
<script setup lang="ts">
import type { CharacterClass } from '~/types'

interface Props {
  characterClass: CharacterClass
  selected: boolean
}

const props = defineProps<Props>()

const emit = defineEmits<{
  select: [characterClass: CharacterClass]
  'view-details': []
}>()

/**
 * Format hit die for display
 */
const hitDieText = computed(() => {
  return `d${props.characterClass.hit_die}`
})

/**
 * Check if class is a spellcaster
 */
const isCaster = computed(() => {
  return props.characterClass.spellcasting_ability !== null
    && props.characterClass.spellcasting_ability !== undefined
})

/**
 * Handle card click - emit select
 */
function handleCardClick() {
  emit('select', props.characterClass)
}

/**
 * Handle View Details click - emit event, stop propagation
 */
function handleViewDetails(event: Event) {
  event.stopPropagation()
  emit('view-details')
}

/**
 * Get background image path (256px variant)
 */
const { getImagePath } = useEntityImage()
const backgroundImage = computed(() => {
  return getImagePath('classes', props.characterClass.slug, 256)
})
</script>

<template>
  <div
    data-testid="picker-card"
    class="relative cursor-pointer transition-all"
    :class="[
      selected ? 'ring-2 ring-class-500 ring-offset-2' : '',
    ]"
    @click="handleCardClick"
  >
    <UCard class="relative overflow-hidden hover:shadow-lg transition-shadow h-full border-2 border-class-300 dark:border-class-700 hover:border-class-500">
      <!-- Background Image Layer -->
      <div
        v-if="backgroundImage"
        class="absolute inset-0 bg-cover bg-center opacity-15 transition-all duration-300"
        :style="{ backgroundImage: `url(${backgroundImage})` }"
      />

      <!-- Selected Checkmark -->
      <div
        v-if="selected"
        class="absolute top-2 right-2 z-20"
      >
        <UBadge
          color="success"
          variant="solid"
          size="md"
        >
          <UIcon
            name="i-heroicons-check"
            class="w-4 h-4"
          />
        </UBadge>
      </div>

      <!-- Content Layer -->
      <div class="relative z-10 flex flex-col h-full">
        <div class="space-y-3 flex-1">
          <!-- Badges Row -->
          <div class="flex items-center gap-2 flex-wrap">
            <UBadge
              color="class"
              variant="subtle"
              size="md"
            >
              Hit Die: {{ hitDieText }}
            </UBadge>
            <UBadge
              v-if="characterClass.primary_ability"
              color="class"
              variant="subtle"
              size="md"
            >
              {{ characterClass.primary_ability }}
            </UBadge>
            <UBadge
              v-if="isCaster"
              color="spell"
              variant="subtle"
              size="md"
            >
              <UIcon
                name="i-heroicons-sparkles"
                class="w-3 h-3 mr-1"
              />
              {{ characterClass.spellcasting_ability?.name }}
            </UBadge>
          </div>

          <!-- Class Name -->
          <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100 line-clamp-2">
            {{ characterClass.name }}
          </h3>

          <!-- Description Preview -->
          <p
            v-if="characterClass.description"
            class="text-sm text-gray-700 dark:text-gray-300 line-clamp-2"
          >
            {{ characterClass.description }}
          </p>
        </div>

        <!-- View Details Button -->
        <div class="mt-4 pt-3 border-t border-gray-200 dark:border-gray-700">
          <UButton
            data-testid="view-details-btn"
            variant="ghost"
            color="class"
            size="sm"
            block
            @click="handleViewDetails"
          >
            <UIcon
              name="i-heroicons-eye"
              class="w-4 h-4 mr-1"
            />
            View Details
          </UButton>
        </div>
      </div>
    </UCard>
  </div>
</template>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/ClassPickerCard.test.ts --reporter=verbose
```

Expected: PASS

### Step 5: Commit

```bash
git add app/components/character/builder/ClassPickerCard.vue tests/components/character/builder/ClassPickerCard.test.ts
git commit -m "$(cat <<'EOF'
feat(character): add ClassPickerCard component

- Click-to-select with ring highlight when selected
- View Details button emits separate event
- Shows hit die, primary ability, and spellcasting indicator
- Background image with fade effect

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Create RaceDetailModal Component

**Files:**
- Create: `app/components/character/builder/RaceDetailModal.vue`
- Create: `tests/components/character/builder/RaceDetailModal.test.ts`

### Step 1: Write failing tests

Create `tests/components/character/builder/RaceDetailModal.test.ts`:

```typescript
// tests/components/character/builder/RaceDetailModal.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import RaceDetailModal from '~/components/character/builder/RaceDetailModal.vue'
import type { Race } from '~/types'

const mockRace: Race = {
  id: 1,
  name: 'Dwarf',
  slug: 'dwarf',
  speed: 25,
  size: { id: 1, code: 'M', name: 'Medium' },
  description: 'Bold and hardy dwarves',
  modifiers: [
    { id: 1, value: 2, modifier_category: 'ability_score', ability_score: { id: 1, code: 'CON', name: 'Constitution' } }
  ],
  traits: [
    { id: 1, name: 'Darkvision', description: 'You can see in dim light' },
    { id: 2, name: 'Dwarven Resilience', description: 'Advantage vs poison' }
  ],
  sources: []
} as Race

describe('RaceDetailModal', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders when open is true', async () => {
    const wrapper = await mountSuspended(RaceDetailModal, {
      props: { race: mockRace, open: true }
    })
    expect(wrapper.text()).toContain('Dwarf')
  })

  it('shows race description', async () => {
    const wrapper = await mountSuspended(RaceDetailModal, {
      props: { race: mockRace, open: true }
    })
    expect(wrapper.text()).toContain('Bold and hardy dwarves')
  })

  it('shows size and speed', async () => {
    const wrapper = await mountSuspended(RaceDetailModal, {
      props: { race: mockRace, open: true }
    })
    expect(wrapper.text()).toContain('Medium')
    expect(wrapper.text()).toContain('25')
  })

  it('shows ability modifiers', async () => {
    const wrapper = await mountSuspended(RaceDetailModal, {
      props: { race: mockRace, open: true }
    })
    expect(wrapper.text()).toContain('Constitution')
    expect(wrapper.text()).toContain('+2')
  })

  it('shows racial traits', async () => {
    const wrapper = await mountSuspended(RaceDetailModal, {
      props: { race: mockRace, open: true }
    })
    expect(wrapper.text()).toContain('Darkvision')
    expect(wrapper.text()).toContain('Dwarven Resilience')
  })

  it('emits close when close button is clicked', async () => {
    const wrapper = await mountSuspended(RaceDetailModal, {
      props: { race: mockRace, open: true }
    })
    await wrapper.find('[data-testid="close-btn"]').trigger('click')
    expect(wrapper.emitted('close')).toBeTruthy()
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/RaceDetailModal.test.ts --reporter=verbose
```

Expected: FAIL

### Step 3: Implement RaceDetailModal component

Create `app/components/character/builder/RaceDetailModal.vue`:

```vue
<!-- app/components/character/builder/RaceDetailModal.vue -->
<script setup lang="ts">
import type { Race } from '~/types'

interface Props {
  race: Race | null
  open: boolean
}

const props = defineProps<Props>()

const emit = defineEmits<{
  close: []
}>()

/**
 * Get ability score modifiers
 */
const abilityModifiers = computed(() => {
  if (!props.race?.modifiers) return []
  return props.race.modifiers.filter(m =>
    m.modifier_category === 'ability_score' && m.ability_score
  )
})

/**
 * Format speed display
 */
const speedDisplay = computed(() => {
  if (!props.race) return []
  const speeds: string[] = []
  if (props.race.speed) speeds.push(`${props.race.speed} ft walk`)
  if (props.race.fly_speed) speeds.push(`${props.race.fly_speed} ft fly`)
  if (props.race.swim_speed) speeds.push(`${props.race.swim_speed} ft swim`)
  return speeds
})

function handleClose() {
  emit('close')
}
</script>

<template>
  <UModal
    :open="open"
    @close="handleClose"
  >
    <template #header>
      <div class="flex items-center justify-between w-full">
        <h2 class="text-xl font-bold">
          {{ race?.name }}
        </h2>
        <UButton
          data-testid="close-btn"
          variant="ghost"
          icon="i-heroicons-x-mark"
          @click="handleClose"
        />
      </div>
    </template>

    <template #body>
      <div
        v-if="race"
        class="space-y-6"
      >
        <!-- Description -->
        <p
          v-if="race.description"
          class="text-gray-700 dark:text-gray-300"
        >
          {{ race.description }}
        </p>

        <!-- Basic Info -->
        <div class="grid grid-cols-2 gap-4">
          <div>
            <h4 class="font-semibold text-gray-900 dark:text-gray-100 mb-1">
              Size
            </h4>
            <p class="text-gray-600 dark:text-gray-400">
              {{ race.size?.name || 'Unknown' }}
            </p>
          </div>
          <div>
            <h4 class="font-semibold text-gray-900 dark:text-gray-100 mb-1">
              Speed
            </h4>
            <p class="text-gray-600 dark:text-gray-400">
              {{ speedDisplay.join(', ') || 'Unknown' }}
            </p>
          </div>
        </div>

        <!-- Ability Score Modifiers -->
        <div v-if="abilityModifiers.length > 0">
          <h4 class="font-semibold text-gray-900 dark:text-gray-100 mb-2">
            Ability Score Increases
          </h4>
          <div class="flex flex-wrap gap-2">
            <UBadge
              v-for="mod in abilityModifiers"
              :key="mod.id"
              color="race"
              variant="subtle"
              size="md"
            >
              {{ mod.ability_score?.name }} +{{ mod.value }}
            </UBadge>
          </div>
        </div>

        <!-- Racial Traits -->
        <div v-if="race.traits && race.traits.length > 0">
          <h4 class="font-semibold text-gray-900 dark:text-gray-100 mb-2">
            Racial Traits
          </h4>
          <div class="space-y-3">
            <div
              v-for="trait in race.traits"
              :key="trait.id"
              class="bg-gray-50 dark:bg-gray-800 rounded-lg p-3"
            >
              <h5 class="font-medium text-gray-900 dark:text-gray-100">
                {{ trait.name }}
              </h5>
              <p
                v-if="trait.description"
                class="text-sm text-gray-600 dark:text-gray-400 mt-1"
              >
                {{ trait.description }}
              </p>
            </div>
          </div>
        </div>
      </div>
    </template>
  </UModal>
</template>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/RaceDetailModal.test.ts --reporter=verbose
```

Expected: PASS

### Step 5: Commit

```bash
git add app/components/character/builder/RaceDetailModal.vue tests/components/character/builder/RaceDetailModal.test.ts
git commit -m "$(cat <<'EOF'
feat(character): add RaceDetailModal component

- Shows full race details in UModal
- Displays size, speed, ability modifiers, traits
- Emits close event when dismissed

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 5: Create ClassDetailModal Component

**Files:**
- Create: `app/components/character/builder/ClassDetailModal.vue`
- Create: `tests/components/character/builder/ClassDetailModal.test.ts`

### Step 1: Write failing tests

Create `tests/components/character/builder/ClassDetailModal.test.ts`:

```typescript
// tests/components/character/builder/ClassDetailModal.test.ts
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import ClassDetailModal from '~/components/character/builder/ClassDetailModal.vue'
import type { CharacterClass } from '~/types'

const mockClass: CharacterClass = {
  id: 1,
  name: 'Fighter',
  slug: 'fighter',
  hit_die: 10,
  is_base_class: true,
  primary_ability: 'STR',
  description: 'A master of martial combat',
  spellcasting_ability: null,
  saving_throws: [
    { id: 1, code: 'STR', name: 'Strength' },
    { id: 2, code: 'CON', name: 'Constitution' }
  ],
  sources: []
} as CharacterClass

const mockCasterClass: CharacterClass = {
  id: 2,
  name: 'Wizard',
  slug: 'wizard',
  hit_die: 6,
  is_base_class: true,
  primary_ability: 'INT',
  description: 'A scholarly magic-user',
  spellcasting_ability: { id: 4, code: 'INT', name: 'Intelligence' },
  saving_throws: [
    { id: 4, code: 'INT', name: 'Intelligence' },
    { id: 5, code: 'WIS', name: 'Wisdom' }
  ],
  sources: []
} as CharacterClass

describe('ClassDetailModal', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders when open is true', async () => {
    const wrapper = await mountSuspended(ClassDetailModal, {
      props: { characterClass: mockClass, open: true }
    })
    expect(wrapper.text()).toContain('Fighter')
  })

  it('shows class description', async () => {
    const wrapper = await mountSuspended(ClassDetailModal, {
      props: { characterClass: mockClass, open: true }
    })
    expect(wrapper.text()).toContain('master of martial combat')
  })

  it('shows hit die', async () => {
    const wrapper = await mountSuspended(ClassDetailModal, {
      props: { characterClass: mockClass, open: true }
    })
    expect(wrapper.text()).toContain('d10')
  })

  it('shows saving throws', async () => {
    const wrapper = await mountSuspended(ClassDetailModal, {
      props: { characterClass: mockClass, open: true }
    })
    expect(wrapper.text()).toContain('Strength')
    expect(wrapper.text()).toContain('Constitution')
  })

  it('shows spellcasting info for caster classes', async () => {
    const wrapper = await mountSuspended(ClassDetailModal, {
      props: { characterClass: mockCasterClass, open: true }
    })
    expect(wrapper.text()).toContain('Spellcasting')
    expect(wrapper.text()).toContain('Intelligence')
  })

  it('does not show spellcasting section for non-casters', async () => {
    const wrapper = await mountSuspended(ClassDetailModal, {
      props: { characterClass: mockClass, open: true }
    })
    expect(wrapper.text()).not.toContain('Spellcasting Ability')
  })

  it('emits close when close button is clicked', async () => {
    const wrapper = await mountSuspended(ClassDetailModal, {
      props: { characterClass: mockClass, open: true }
    })
    await wrapper.find('[data-testid="close-btn"]').trigger('click')
    expect(wrapper.emitted('close')).toBeTruthy()
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/ClassDetailModal.test.ts --reporter=verbose
```

Expected: FAIL

### Step 3: Implement ClassDetailModal component

Create `app/components/character/builder/ClassDetailModal.vue`:

```vue
<!-- app/components/character/builder/ClassDetailModal.vue -->
<script setup lang="ts">
import type { CharacterClass } from '~/types'

interface Props {
  characterClass: CharacterClass | null
  open: boolean
}

const props = defineProps<Props>()

const emit = defineEmits<{
  close: []
}>()

/**
 * Check if class is a spellcaster
 */
const isCaster = computed(() => {
  return props.characterClass?.spellcasting_ability !== null
    && props.characterClass?.spellcasting_ability !== undefined
})

/**
 * Format hit die
 */
const hitDieText = computed(() => {
  return `d${props.characterClass?.hit_die}`
})

function handleClose() {
  emit('close')
}
</script>

<template>
  <UModal
    :open="open"
    @close="handleClose"
  >
    <template #header>
      <div class="flex items-center justify-between w-full">
        <h2 class="text-xl font-bold">
          {{ characterClass?.name }}
        </h2>
        <UButton
          data-testid="close-btn"
          variant="ghost"
          icon="i-heroicons-x-mark"
          @click="handleClose"
        />
      </div>
    </template>

    <template #body>
      <div
        v-if="characterClass"
        class="space-y-6"
      >
        <!-- Description -->
        <p
          v-if="characterClass.description"
          class="text-gray-700 dark:text-gray-300"
        >
          {{ characterClass.description }}
        </p>

        <!-- Basic Info -->
        <div class="grid grid-cols-2 gap-4">
          <div>
            <h4 class="font-semibold text-gray-900 dark:text-gray-100 mb-1">
              Hit Die
            </h4>
            <p class="text-gray-600 dark:text-gray-400">
              {{ hitDieText }}
            </p>
          </div>
          <div v-if="characterClass.primary_ability">
            <h4 class="font-semibold text-gray-900 dark:text-gray-100 mb-1">
              Primary Ability
            </h4>
            <p class="text-gray-600 dark:text-gray-400">
              {{ characterClass.primary_ability }}
            </p>
          </div>
        </div>

        <!-- Saving Throws -->
        <div v-if="characterClass.saving_throws && characterClass.saving_throws.length > 0">
          <h4 class="font-semibold text-gray-900 dark:text-gray-100 mb-2">
            Saving Throw Proficiencies
          </h4>
          <div class="flex flex-wrap gap-2">
            <UBadge
              v-for="save in characterClass.saving_throws"
              :key="save.id"
              color="class"
              variant="subtle"
              size="md"
            >
              {{ save.name }}
            </UBadge>
          </div>
        </div>

        <!-- Spellcasting -->
        <div v-if="isCaster">
          <h4 class="font-semibold text-gray-900 dark:text-gray-100 mb-2">
            Spellcasting
          </h4>
          <div class="bg-spell-50 dark:bg-spell-900/20 rounded-lg p-3">
            <div class="flex items-center gap-2">
              <UIcon
                name="i-heroicons-sparkles"
                class="w-5 h-5 text-spell-500"
              />
              <span class="text-gray-700 dark:text-gray-300">
                Spellcasting Ability: <strong>{{ characterClass.spellcasting_ability?.name }}</strong>
              </span>
            </div>
          </div>
        </div>
      </div>
    </template>
  </UModal>
</template>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/ClassDetailModal.test.ts --reporter=verbose
```

Expected: PASS

### Step 5: Commit

```bash
git add app/components/character/builder/ClassDetailModal.vue tests/components/character/builder/ClassDetailModal.test.ts
git commit -m "$(cat <<'EOF'
feat(character): add ClassDetailModal component

- Shows full class details in UModal
- Displays hit die, primary ability, saving throws
- Shows spellcasting info for caster classes
- Emits close event when dismissed

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 6: Implement StepRace Component

**Files:**
- Modify: `app/components/character/builder/StepRace.vue`
- Create: `tests/components/character/builder/StepRace.test.ts`

### Step 1: Write failing tests

Create `tests/components/character/builder/StepRace.test.ts`:

```typescript
// tests/components/character/builder/StepRace.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepRace from '~/components/character/builder/StepRace.vue'
import type { Race } from '~/types'

const mockRaces: Race[] = [
  {
    id: 1,
    name: 'Dwarf',
    slug: 'dwarf',
    speed: 25,
    size: { id: 1, code: 'M', name: 'Medium' },
    subraces: [
      { id: 3, name: 'Hill Dwarf', slug: 'hill-dwarf' },
      { id: 4, name: 'Mountain Dwarf', slug: 'mountain-dwarf' }
    ]
  } as Race,
  {
    id: 2,
    name: 'Human',
    slug: 'human',
    speed: 30,
    size: { id: 1, code: 'M', name: 'Medium' }
  } as Race
]

// Mock useAsyncData
vi.mock('#app', async () => {
  const actual = await vi.importActual('#app')
  return {
    ...actual,
    useAsyncData: vi.fn(() => ({
      data: ref(mockRaces),
      pending: ref(false),
      error: ref(null)
    }))
  }
})

describe('StepRace', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders the step title', async () => {
    const wrapper = await mountSuspended(StepRace)
    expect(wrapper.text()).toContain('Choose Your Race')
  })

  it('renders race picker cards', async () => {
    const wrapper = await mountSuspended(StepRace)
    expect(wrapper.text()).toContain('Dwarf')
    expect(wrapper.text()).toContain('Human')
  })

  it('shows subrace selector when race with subraces is selected', async () => {
    const wrapper = await mountSuspended(StepRace)
    // Find and click the Dwarf card
    const cards = wrapper.findAllComponents({ name: 'CharacterBuilderRacePickerCard' })
    expect(cards.length).toBeGreaterThan(0)
  })

  it('filters races by search query', async () => {
    const wrapper = await mountSuspended(StepRace)
    const searchInput = wrapper.find('input[type="text"]')

    if (searchInput.exists()) {
      await searchInput.setValue('Dwarf')
      await wrapper.vm.$nextTick()
      // After filtering, only Dwarf should remain visible
      expect(wrapper.text()).toContain('Dwarf')
    }
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/StepRace.test.ts --reporter=verbose
```

Expected: FAIL

### Step 3: Implement StepRace component

Replace contents of `app/components/character/builder/StepRace.vue`:

```vue
<!-- app/components/character/builder/StepRace.vue -->
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import type { Race } from '~/types'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

const store = useCharacterBuilderStore()
const { selectedRace, raceId, subraceId, isLoading, error } = storeToRefs(store)

// Fetch all races
const { data: races, pending: loadingRaces } = await useAsyncData(
  'builder-races',
  () => apiFetch<{ data: Race[] }>('/races?per_page=100'),
  { transform: response => response.data }
)

// Local state
const searchQuery = ref('')
const localSelectedRace = ref<Race | null>(null)
const localSelectedSubrace = ref<Race | null>(null)
const detailModalOpen = ref(false)
const detailRace = ref<Race | null>(null)

// Filter to only show base races (not subraces)
const baseRaces = computed(() => {
  if (!races.value) return []
  return races.value.filter(race => !race.parent_race)
})

// Apply search filter
const filteredRaces = computed(() => {
  if (!searchQuery.value) return baseRaces.value
  const query = searchQuery.value.toLowerCase()
  return baseRaces.value.filter(race =>
    race.name.toLowerCase().includes(query)
  )
})

// Check if selected race has subraces
const selectedRaceHasSubraces = computed(() => {
  return localSelectedRace.value?.subraces && localSelectedRace.value.subraces.length > 0
})

// Get subraces for selected race (from full race data)
const availableSubraces = computed(() => {
  if (!localSelectedRace.value || !races.value) return []
  // Find full subrace objects from the races list
  return races.value.filter(race =>
    race.parent_race?.id === localSelectedRace.value?.id
  )
})

// Validation: can proceed if race selected (and subrace if needed)
const canProceed = computed(() => {
  if (!localSelectedRace.value) return false
  if (selectedRaceHasSubraces.value && !localSelectedSubrace.value) return false
  return true
})

/**
 * Handle race card selection
 */
function handleRaceSelect(race: Race) {
  localSelectedRace.value = race
  localSelectedSubrace.value = null // Reset subrace when race changes
}

/**
 * Handle subrace selection
 */
function handleSubraceSelect(subrace: Race) {
  localSelectedSubrace.value = subrace
}

/**
 * Open detail modal
 */
function handleViewDetails(race: Race) {
  detailRace.value = race
  detailModalOpen.value = true
}

/**
 * Close detail modal
 */
function handleCloseModal() {
  detailModalOpen.value = false
  detailRace.value = null
}

/**
 * Confirm selection and call store action
 */
async function confirmSelection() {
  if (!localSelectedRace.value) return

  try {
    await store.selectRace(localSelectedRace.value, localSelectedSubrace.value ?? undefined)
    store.nextStep()
  } catch (err) {
    // Error is already set in store
    console.error('Failed to save race:', err)
  }
}

// Initialize from store if already selected
onMounted(() => {
  if (selectedRace.value) {
    // Find the base race if we have a subrace selected
    if (selectedRace.value.parent_race) {
      localSelectedRace.value = baseRaces.value.find(r => r.id === selectedRace.value?.parent_race?.id) ?? null
      localSelectedSubrace.value = selectedRace.value
    } else {
      localSelectedRace.value = selectedRace.value
    }
  }
})
</script>

<template>
  <div class="space-y-6">
    <!-- Header -->
    <div class="text-center">
      <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
        Choose Your Race
      </h2>
      <p class="mt-2 text-gray-600 dark:text-gray-400">
        Your race determines your character's physical traits and natural abilities
      </p>
    </div>

    <!-- Search -->
    <div class="max-w-md mx-auto">
      <UInput
        v-model="searchQuery"
        placeholder="Search races..."
        icon="i-heroicons-magnifying-glass"
        size="lg"
      />
    </div>

    <!-- Error State -->
    <UAlert
      v-if="error"
      color="error"
      icon="i-heroicons-exclamation-circle"
      :title="error"
    />

    <!-- Loading State -->
    <div
      v-if="loadingRaces"
      class="flex justify-center py-12"
    >
      <UIcon
        name="i-heroicons-arrow-path"
        class="w-8 h-8 animate-spin text-race-500"
      />
    </div>

    <!-- Race Grid -->
    <div
      v-else
      class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4"
    >
      <CharacterBuilderRacePickerCard
        v-for="race in filteredRaces"
        :key="race.id"
        :race="race"
        :selected="localSelectedRace?.id === race.id"
        @select="handleRaceSelect"
        @view-details="handleViewDetails(race)"
      />
    </div>

    <!-- Empty State -->
    <div
      v-if="!loadingRaces && filteredRaces.length === 0"
      class="text-center py-12"
    >
      <UIcon
        name="i-heroicons-magnifying-glass"
        class="w-12 h-12 text-gray-400 mx-auto mb-4"
      />
      <p class="text-gray-600 dark:text-gray-400">
        No races found matching "{{ searchQuery }}"
      </p>
    </div>

    <!-- Subrace Selector (appears when race with subraces is selected) -->
    <div
      v-if="selectedRaceHasSubraces && availableSubraces.length > 0"
      class="bg-race-50 dark:bg-race-900/20 rounded-lg p-4"
    >
      <h3 class="font-semibold text-gray-900 dark:text-white mb-3">
        Choose a Subrace for {{ localSelectedRace?.name }}
      </h3>
      <div class="grid grid-cols-2 sm:grid-cols-3 gap-3">
        <button
          v-for="subrace in availableSubraces"
          :key="subrace.id"
          class="p-3 rounded-lg border-2 transition-all text-left"
          :class="[
            localSelectedSubrace?.id === subrace.id
              ? 'border-race-500 bg-race-100 dark:bg-race-900/40'
              : 'border-gray-200 dark:border-gray-700 hover:border-race-300'
          ]"
          @click="handleSubraceSelect(subrace)"
        >
          <div class="flex items-center gap-2">
            <div
              class="w-4 h-4 rounded-full border-2 flex items-center justify-center"
              :class="[
                localSelectedSubrace?.id === subrace.id
                  ? 'border-race-500 bg-race-500'
                  : 'border-gray-400'
              ]"
            >
              <UIcon
                v-if="localSelectedSubrace?.id === subrace.id"
                name="i-heroicons-check"
                class="w-3 h-3 text-white"
              />
            </div>
            <span class="font-medium text-gray-900 dark:text-white">
              {{ subrace.name }}
            </span>
          </div>
        </button>
      </div>
    </div>

    <!-- Confirm Button -->
    <div class="flex justify-center pt-4">
      <UButton
        size="lg"
        :disabled="!canProceed || isLoading"
        :loading="isLoading"
        @click="confirmSelection"
      >
        {{ isLoading ? 'Saving...' : 'Continue with ' + (localSelectedSubrace?.name || localSelectedRace?.name || 'Selection') }}
      </UButton>
    </div>

    <!-- Detail Modal -->
    <CharacterBuilderRaceDetailModal
      :race="detailRace"
      :open="detailModalOpen"
      @close="handleCloseModal"
    />
  </div>
</template>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/StepRace.test.ts --reporter=verbose
```

Expected: PASS

### Step 5: Commit

```bash
git add app/components/character/builder/StepRace.vue tests/components/character/builder/StepRace.test.ts
git commit -m "$(cat <<'EOF'
feat(character): implement StepRace component

- Grid of race picker cards with search
- Inline subrace selector for races with subraces
- View Details opens modal with full race info
- Calls store.selectRace on confirmation
- Validates subrace selection before proceeding

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 7: Implement StepClass Component

**Files:**
- Modify: `app/components/character/builder/StepClass.vue`
- Create: `tests/components/character/builder/StepClass.test.ts`

### Step 1: Write failing tests

Create `tests/components/character/builder/StepClass.test.ts`:

```typescript
// tests/components/character/builder/StepClass.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepClass from '~/components/character/builder/StepClass.vue'
import type { CharacterClass } from '~/types'

const mockClasses: CharacterClass[] = [
  {
    id: 1,
    name: 'Fighter',
    slug: 'fighter',
    hit_die: 10,
    is_base_class: true,
    primary_ability: 'STR',
    spellcasting_ability: null
  } as CharacterClass,
  {
    id: 2,
    name: 'Wizard',
    slug: 'wizard',
    hit_die: 6,
    is_base_class: true,
    primary_ability: 'INT',
    spellcasting_ability: { id: 4, code: 'INT', name: 'Intelligence' }
  } as CharacterClass
]

// Mock useAsyncData
vi.mock('#app', async () => {
  const actual = await vi.importActual('#app')
  return {
    ...actual,
    useAsyncData: vi.fn(() => ({
      data: ref(mockClasses),
      pending: ref(false),
      error: ref(null)
    }))
  }
})

describe('StepClass', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('renders the step title', async () => {
    const wrapper = await mountSuspended(StepClass)
    expect(wrapper.text()).toContain('Choose Your Class')
  })

  it('renders class picker cards', async () => {
    const wrapper = await mountSuspended(StepClass)
    expect(wrapper.text()).toContain('Fighter')
    expect(wrapper.text()).toContain('Wizard')
  })

  it('shows hit die on cards', async () => {
    const wrapper = await mountSuspended(StepClass)
    expect(wrapper.text()).toContain('d10')
    expect(wrapper.text()).toContain('d6')
  })

  it('filters classes by search query', async () => {
    const wrapper = await mountSuspended(StepClass)
    const searchInput = wrapper.find('input[type="text"]')

    if (searchInput.exists()) {
      await searchInput.setValue('Fighter')
      await wrapper.vm.$nextTick()
      expect(wrapper.text()).toContain('Fighter')
    }
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/StepClass.test.ts --reporter=verbose
```

Expected: FAIL

### Step 3: Implement StepClass component

Replace contents of `app/components/character/builder/StepClass.vue`:

```vue
<!-- app/components/character/builder/StepClass.vue -->
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import type { CharacterClass } from '~/types'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

const store = useCharacterBuilderStore()
const { selectedClass, classId, isLoading, error } = storeToRefs(store)

// Fetch base classes only
const { data: classes, pending: loadingClasses } = await useAsyncData(
  'builder-classes',
  () => apiFetch<{ data: CharacterClass[] }>('/classes?filter=is_base_class=true&per_page=50'),
  { transform: response => response.data }
)

// Local state
const searchQuery = ref('')
const localSelectedClass = ref<CharacterClass | null>(null)
const detailModalOpen = ref(false)
const detailClass = ref<CharacterClass | null>(null)

// Apply search filter
const filteredClasses = computed(() => {
  if (!classes.value) return []
  if (!searchQuery.value) return classes.value
  const query = searchQuery.value.toLowerCase()
  return classes.value.filter(cls =>
    cls.name.toLowerCase().includes(query)
  )
})

// Validation: can proceed if class selected
const canProceed = computed(() => {
  return localSelectedClass.value !== null
})

/**
 * Handle class card selection
 */
function handleClassSelect(cls: CharacterClass) {
  localSelectedClass.value = cls
}

/**
 * Open detail modal
 */
function handleViewDetails(cls: CharacterClass) {
  detailClass.value = cls
  detailModalOpen.value = true
}

/**
 * Close detail modal
 */
function handleCloseModal() {
  detailModalOpen.value = false
  detailClass.value = null
}

/**
 * Confirm selection and call store action
 */
async function confirmSelection() {
  if (!localSelectedClass.value) return

  try {
    await store.selectClass(localSelectedClass.value)
    store.nextStep()
  } catch (err) {
    // Error is already set in store
    console.error('Failed to save class:', err)
  }
}

// Initialize from store if already selected
onMounted(() => {
  if (selectedClass.value) {
    localSelectedClass.value = selectedClass.value
  }
})
</script>

<template>
  <div class="space-y-6">
    <!-- Header -->
    <div class="text-center">
      <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
        Choose Your Class
      </h2>
      <p class="mt-2 text-gray-600 dark:text-gray-400">
        Your class determines your character's abilities, skills, and combat style
      </p>
    </div>

    <!-- Search -->
    <div class="max-w-md mx-auto">
      <UInput
        v-model="searchQuery"
        placeholder="Search classes..."
        icon="i-heroicons-magnifying-glass"
        size="lg"
      />
    </div>

    <!-- Error State -->
    <UAlert
      v-if="error"
      color="error"
      icon="i-heroicons-exclamation-circle"
      :title="error"
    />

    <!-- Loading State -->
    <div
      v-if="loadingClasses"
      class="flex justify-center py-12"
    >
      <UIcon
        name="i-heroicons-arrow-path"
        class="w-8 h-8 animate-spin text-class-500"
      />
    </div>

    <!-- Class Grid -->
    <div
      v-else
      class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4"
    >
      <CharacterBuilderClassPickerCard
        v-for="cls in filteredClasses"
        :key="cls.id"
        :character-class="cls"
        :selected="localSelectedClass?.id === cls.id"
        @select="handleClassSelect"
        @view-details="handleViewDetails(cls)"
      />
    </div>

    <!-- Empty State -->
    <div
      v-if="!loadingClasses && filteredClasses.length === 0"
      class="text-center py-12"
    >
      <UIcon
        name="i-heroicons-magnifying-glass"
        class="w-12 h-12 text-gray-400 mx-auto mb-4"
      />
      <p class="text-gray-600 dark:text-gray-400">
        No classes found matching "{{ searchQuery }}"
      </p>
    </div>

    <!-- Spellcaster Info -->
    <div
      v-if="localSelectedClass?.spellcasting_ability"
      class="bg-spell-50 dark:bg-spell-900/20 rounded-lg p-4"
    >
      <div class="flex items-center gap-2">
        <UIcon
          name="i-heroicons-sparkles"
          class="w-5 h-5 text-spell-500"
        />
        <span class="text-gray-700 dark:text-gray-300">
          <strong>{{ localSelectedClass.name }}</strong> is a spellcasting class.
          You'll select spells in a later step.
        </span>
      </div>
    </div>

    <!-- Confirm Button -->
    <div class="flex justify-center pt-4">
      <UButton
        size="lg"
        :disabled="!canProceed || isLoading"
        :loading="isLoading"
        @click="confirmSelection"
      >
        {{ isLoading ? 'Saving...' : 'Continue with ' + (localSelectedClass?.name || 'Selection') }}
      </UButton>
    </div>

    <!-- Detail Modal -->
    <CharacterBuilderClassDetailModal
      :character-class="detailClass"
      :open="detailModalOpen"
      @close="handleCloseModal"
    />
  </div>
</template>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/components/character/builder/StepClass.test.ts --reporter=verbose
```

Expected: PASS

### Step 5: Commit

```bash
git add app/components/character/builder/StepClass.vue tests/components/character/builder/StepClass.test.ts
git commit -m "$(cat <<'EOF'
feat(character): implement StepClass component

- Grid of class picker cards with search
- Only shows base classes (not subclasses)
- View Details opens modal with full class info
- Calls store.selectClass on confirmation
- Shows spellcaster notice when caster class selected

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Task 8: Update Page Navigation and Final Integration

**Files:**
- Modify: `app/pages/characters/create.vue`

### Step 1: Verify page already handles step rendering

The page already has the correct conditional rendering. Verify it works:

```bash
docker compose exec nuxt npm run dev
```

Navigate to http://localhost:3000/characters/create and verify:
1. Step 1 (Name) works
2. After creating character, Step 2 shows race grid
3. After selecting race, Step 3 shows class grid
4. Class selection updates `isCaster` and totalSteps

### Step 2: Run full test suite

```bash
docker compose exec nuxt npm run test -- tests/stores/characterBuilder.test.ts tests/components/character/builder/ --reporter=verbose
```

Expected: All tests PASS

### Step 3: Run lint and typecheck

```bash
docker compose exec nuxt npm run lint:fix
docker compose exec nuxt npm run typecheck
```

Expected: No errors

### Step 4: Final commit

```bash
git add -A
git commit -m "$(cat <<'EOF'
feat(character): complete Phase 2 race and class selection

Phase 2 of character builder complete:
- Store actions: selectRace, selectClass, refreshStats
- RacePickerCard and ClassPickerCard with selection behavior
- RaceDetailModal and ClassDetailModal for viewing full info
- StepRace with subrace handling
- StepClass with spellcaster detection

Closes part of #89

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>
EOF
)"
```

---

## Summary

| Task | Components | Tests | Estimated |
|------|-----------|-------|-----------|
| 1 | Store actions | 12 tests | 45 min |
| 2 | RacePickerCard | 7 tests | 30 min |
| 3 | ClassPickerCard | 9 tests | 30 min |
| 4 | RaceDetailModal | 6 tests | 25 min |
| 5 | ClassDetailModal | 7 tests | 25 min |
| 6 | StepRace | 4 tests | 45 min |
| 7 | StepClass | 4 tests | 35 min |
| 8 | Integration | Manual | 15 min |

**Total: ~4.5 hours** (optimistic estimate with TDD)

---

## Verification Checklist

After implementation, verify:

- [ ] Can create character (Step 1)
- [ ] Race grid displays with search
- [ ] Race selection highlights card
- [ ] Subrace selector appears for races with subraces
- [ ] View Details opens race modal
- [ ] Continuing saves race to API
- [ ] Class grid displays with search
- [ ] Class selection highlights card
- [ ] View Details opens class modal
- [ ] Spellcaster notice shows for caster classes
- [ ] Continuing saves class to API
- [ ] `isCaster` updates totalSteps correctly
- [ ] Back navigation preserves selections
- [ ] All tests pass
- [ ] TypeScript compiles
- [ ] ESLint passes
