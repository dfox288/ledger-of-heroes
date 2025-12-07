# Character Builder Phase 4 Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Complete the Character Builder wizard with Background, Equipment, Spells, and Review steps.

**Architecture:** Four new wizard steps following existing patterns (StepRace/StepClass). Each step has picker cards, detail modals, and store actions. Equipment step combines class + background gear. Spells step handles racial + class spells.

**Tech Stack:** Vue 3, Pinia, TypeScript, Vitest, NuxtUI 4

**Design Doc:** `docs/plans/2025-12-03-character-builder-phase-4-design.md`

**Test Command:** `docker compose exec nuxt npm run test:classes` (includes character builder tests)

---

## Progress Tracker

| Task | Description | Status | Commit |
|------|-------------|--------|--------|
| 1 | Store - Add Background Action | ✅ Complete | `01c6ac2` |
| 2 | Store - Update totalSteps | ✅ Complete | `fe932a5` |
| 3 | BackgroundPickerCard | ✅ Complete | `dd61fcb` |
| 4 | BackgroundDetailModal | ✅ Complete | `d59ca4b` |
| 5 | StepBackground | ✅ Complete | `db31f23` |
| 6 | EquipmentChoiceGroup | ✅ Complete | `8f1b5ee` |
| 7 | Store - Equipment State | ✅ Complete | `916b3bf` |
| 8 | StepEquipment | ✅ Complete | `af47863` |
| 9 | SpellPickerCard | ✅ Complete | `9551f54` |
| 10 | Store - Spell Actions | ✅ Complete | `1096fed` |
| 11 | StepSpells | ✅ Complete | `bf12c36` |
| 12 | StepReview | ✅ Complete | `5d782f7` |
| 13 | Update Wizard Page | ✅ Complete | `31ea55b` |
| 14 | Integration Tests | ✅ Complete | `5d782f7` |

**Last Updated:** 2025-12-03 ✅ **PHASE 4 COMPLETE**

---

## Task 1: Store - Add Background Action

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Test: `tests/stores/characterBuilder.spec.ts`

**Step 1: Write the failing test**

Add to `tests/stores/characterBuilder.spec.ts`:

```typescript
describe('selectBackground', () => {
  it('saves background to API and updates store state', async () => {
    const store = useCharacterBuilderStore()
    store.characterId = 1

    const mockBackground = {
      id: 5,
      slug: 'soldier',
      name: 'Soldier',
      feature_name: 'Military Rank',
      feature_description: 'You have a military rank...',
      proficiencies: [],
      equipment: [],
      languages: []
    }

    // Mock the API call
    const apiFetch = vi.fn().mockResolvedValue({ data: { id: 1 } })
    vi.mocked(useApi).mockReturnValue({ apiFetch } as any)

    await store.selectBackground(mockBackground as any)

    expect(apiFetch).toHaveBeenCalledWith('/characters/1', {
      method: 'PATCH',
      body: { background_id: 5 }
    })
    expect(store.backgroundId).toBe(5)
    expect(store.selectedBackground).toEqual(mockBackground)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.spec.ts -t "selectBackground"`

Expected: FAIL - `selectBackground is not a function`

**Step 3: Write minimal implementation**

In `app/stores/characterBuilder.ts`, add after `saveAbilityScores`:

```typescript
/**
 * Step 5: Select background
 */
async function selectBackground(background: Background): Promise<void> {
  isLoading.value = true
  error.value = null

  try {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: { background_id: background.id }
    })

    backgroundId.value = background.id
    selectedBackground.value = background

    await refreshStats()
  } catch (err: unknown) {
    error.value = 'Failed to save background'
    throw err
  } finally {
    isLoading.value = false
  }
}
```

Add to return statement:
```typescript
selectBackground,
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.spec.ts -t "selectBackground"`

Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.spec.ts
git commit -m "feat(character): add selectBackground store action"
```

---

## Task 2: Store - Update totalSteps Computed

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Test: `tests/stores/characterBuilder.spec.ts`

**Step 1: Write the failing test**

```typescript
describe('totalSteps', () => {
  it('returns 7 for non-casters', () => {
    const store = useCharacterBuilderStore()
    store.selectedClass = {
      id: 1,
      name: 'Fighter',
      spellcasting_ability: null
    } as any

    expect(store.totalSteps).toBe(7)
  })

  it('returns 8 for casters', () => {
    const store = useCharacterBuilderStore()
    store.selectedClass = {
      id: 2,
      name: 'Wizard',
      spellcasting_ability: { id: 4, code: 'INT', name: 'Intelligence' }
    } as any

    expect(store.totalSteps).toBe(8)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.spec.ts -t "totalSteps"`

Expected: FAIL - expects 7, got 6 (or similar)

**Step 3: Write minimal implementation**

Update the `totalSteps` computed in `app/stores/characterBuilder.ts`:

```typescript
const totalSteps = computed(() => isCaster.value ? 8 : 7)
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.spec.ts -t "totalSteps"`

Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.spec.ts
git commit -m "feat(character): update totalSteps for new wizard flow (7/8 steps)"
```

---

## Task 3: BackgroundPickerCard Component

**Files:**
- Create: `app/components/character/builder/BackgroundPickerCard.vue`
- Create: `tests/components/character/builder/BackgroundPickerCard.spec.ts`

**Step 1: Write the failing test**

Create `tests/components/character/builder/BackgroundPickerCard.spec.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import BackgroundPickerCard from '~/components/character/builder/BackgroundPickerCard.vue'

const mockBackground = {
  id: 1,
  slug: 'acolyte',
  name: 'Acolyte',
  feature_name: 'Shelter of the Faithful',
  feature_description: 'As an acolyte, you command respect...',
  proficiencies: [
    { proficiency_type: 'skill', skill: { name: 'Insight' } },
    { proficiency_type: 'skill', skill: { name: 'Religion' } }
  ],
  languages: [
    { language: { name: 'Celestial' } },
    { language: { name: 'Infernal' } }
  ],
  equipment: []
}

describe('BackgroundPickerCard', () => {
  it('displays background name', async () => {
    const wrapper = await mountSuspended(BackgroundPickerCard, {
      props: { background: mockBackground, selected: false }
    })

    expect(wrapper.text()).toContain('Acolyte')
  })

  it('displays feature name badge', async () => {
    const wrapper = await mountSuspended(BackgroundPickerCard, {
      props: { background: mockBackground, selected: false }
    })

    expect(wrapper.text()).toContain('Shelter of the Faithful')
  })

  it('displays skill proficiencies', async () => {
    const wrapper = await mountSuspended(BackgroundPickerCard, {
      props: { background: mockBackground, selected: false }
    })

    expect(wrapper.text()).toContain('Insight')
    expect(wrapper.text()).toContain('Religion')
  })

  it('shows checkmark when selected', async () => {
    const wrapper = await mountSuspended(BackgroundPickerCard, {
      props: { background: mockBackground, selected: true }
    })

    expect(wrapper.find('[data-test="selected-check"]').exists()).toBe(true)
  })

  it('emits select event on click', async () => {
    const wrapper = await mountSuspended(BackgroundPickerCard, {
      props: { background: mockBackground, selected: false }
    })

    await wrapper.find('[data-test="card-button"]').trigger('click')

    expect(wrapper.emitted('select')).toBeTruthy()
    expect(wrapper.emitted('select')![0]).toEqual([mockBackground])
  })

  it('emits viewDetails event on details button click', async () => {
    const wrapper = await mountSuspended(BackgroundPickerCard, {
      props: { background: mockBackground, selected: false }
    })

    await wrapper.find('[data-test="view-details-btn"]').trigger('click')

    expect(wrapper.emitted('viewDetails')).toBeTruthy()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/BackgroundPickerCard.spec.ts`

Expected: FAIL - component not found

**Step 3: Write minimal implementation**

Create `app/components/character/builder/BackgroundPickerCard.vue`:

```vue
<script setup lang="ts">
import type { Background } from '~/types'

interface Props {
  background: Background
  selected: boolean
}

const props = defineProps<Props>()

const emit = defineEmits<{
  select: [background: Background]
  viewDetails: []
}>()

// Extract skill names from proficiencies
const skillNames = computed(() => {
  if (!props.background.proficiencies) return []
  return props.background.proficiencies
    .filter(p => p.proficiency_type === 'skill')
    .map(p => p.skill?.name)
    .filter(Boolean)
})

// Count languages
const languageCount = computed(() => {
  return props.background.languages?.length ?? 0
})

function handleSelect() {
  emit('select', props.background)
}

function handleViewDetails(event: Event) {
  event.stopPropagation()
  emit('viewDetails')
}
</script>

<template>
  <div class="relative">
    <button
      data-test="card-button"
      type="button"
      class="w-full text-left"
      @click="handleSelect"
    >
      <UCard
        class="h-full transition-all"
        :class="[
          selected
            ? 'ring-2 ring-background-500 bg-background-50 dark:bg-background-900/30'
            : 'hover:ring-2 hover:ring-background-300'
        ]"
      >
        <!-- Selected Checkmark -->
        <div
          v-if="selected"
          data-test="selected-check"
          class="absolute top-2 right-2 w-6 h-6 bg-background-500 rounded-full flex items-center justify-center"
        >
          <UIcon
            name="i-heroicons-check"
            class="w-4 h-4 text-white"
          />
        </div>

        <div class="space-y-2">
          <!-- Feature Badge -->
          <UBadge
            v-if="background.feature_name"
            color="background"
            variant="subtle"
            size="md"
          >
            {{ background.feature_name }}
          </UBadge>

          <!-- Name -->
          <h3 class="text-lg font-semibold text-gray-900 dark:text-white">
            {{ background.name }}
          </h3>

          <!-- Skills -->
          <div
            v-if="skillNames.length > 0"
            class="flex items-center gap-1 text-sm text-gray-600 dark:text-gray-400"
          >
            <UIcon
              name="i-heroicons-academic-cap"
              class="w-4 h-4"
            />
            <span>{{ skillNames.join(', ') }}</span>
          </div>

          <!-- Languages -->
          <div
            v-if="languageCount > 0"
            class="flex items-center gap-1 text-sm text-gray-600 dark:text-gray-400"
          >
            <UIcon
              name="i-heroicons-language"
              class="w-4 h-4"
            />
            <span>{{ languageCount }} {{ languageCount === 1 ? 'Language' : 'Languages' }}</span>
          </div>
        </div>

        <!-- View Details Button -->
        <template #footer>
          <UButton
            data-test="view-details-btn"
            variant="ghost"
            color="neutral"
            size="sm"
            block
            @click="handleViewDetails"
          >
            View Details
          </UButton>
        </template>
      </UCard>
    </button>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/BackgroundPickerCard.spec.ts`

Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/BackgroundPickerCard.vue tests/components/character/builder/BackgroundPickerCard.spec.ts
git commit -m "feat(character): add BackgroundPickerCard component"
```

---

## Task 4: BackgroundDetailModal Component

**Files:**
- Create: `app/components/character/builder/BackgroundDetailModal.vue`
- Create: `tests/components/character/builder/BackgroundDetailModal.spec.ts`

**Step 1: Write the failing test**

Create `tests/components/character/builder/BackgroundDetailModal.spec.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import BackgroundDetailModal from '~/components/character/builder/BackgroundDetailModal.vue'

const mockBackground = {
  id: 1,
  slug: 'acolyte',
  name: 'Acolyte',
  feature_name: 'Shelter of the Faithful',
  feature_description: 'As an acolyte, you command respect from those who share your faith.',
  proficiencies: [
    { proficiency_type: 'skill', skill: { name: 'Insight' } },
    { proficiency_type: 'skill', skill: { name: 'Religion' } }
  ],
  languages: [
    { language: { name: 'Celestial' } },
    { language: { name: 'Infernal' } }
  ],
  equipment: [
    { item: { name: 'Holy Symbol' }, quantity: 1, is_choice: false },
    { item: { name: 'Prayer Book' }, quantity: 1, is_choice: false }
  ]
}

describe('BackgroundDetailModal', () => {
  it('displays background name in header', async () => {
    const wrapper = await mountSuspended(BackgroundDetailModal, {
      props: { background: mockBackground, open: true }
    })

    expect(wrapper.text()).toContain('Acolyte')
  })

  it('displays feature name and description', async () => {
    const wrapper = await mountSuspended(BackgroundDetailModal, {
      props: { background: mockBackground, open: true }
    })

    expect(wrapper.text()).toContain('Shelter of the Faithful')
    expect(wrapper.text()).toContain('command respect')
  })

  it('displays skill proficiencies', async () => {
    const wrapper = await mountSuspended(BackgroundDetailModal, {
      props: { background: mockBackground, open: true }
    })

    expect(wrapper.text()).toContain('Insight')
    expect(wrapper.text()).toContain('Religion')
  })

  it('displays equipment list', async () => {
    const wrapper = await mountSuspended(BackgroundDetailModal, {
      props: { background: mockBackground, open: true }
    })

    expect(wrapper.text()).toContain('Holy Symbol')
    expect(wrapper.text()).toContain('Prayer Book')
  })

  it('emits close event when close button clicked', async () => {
    const wrapper = await mountSuspended(BackgroundDetailModal, {
      props: { background: mockBackground, open: true }
    })

    await wrapper.find('[data-test="close-btn"]').trigger('click')

    expect(wrapper.emitted('close')).toBeTruthy()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/BackgroundDetailModal.spec.ts`

Expected: FAIL - component not found

**Step 3: Write minimal implementation**

Create `app/components/character/builder/BackgroundDetailModal.vue`:

```vue
<script setup lang="ts">
import type { Background } from '~/types'

interface Props {
  background: Background | null
  open: boolean
}

const props = defineProps<Props>()

const emit = defineEmits<{
  close: []
}>()

// Use background stats composable
const backgroundRef = computed(() => props.background)
const {
  skillProficiencies,
  toolProficiencies,
  languages,
  equipmentCount,
  startingGold
} = useBackgroundStats(backgroundRef)

// Get equipment items (excluding gold)
const equipmentItems = computed(() => {
  if (!props.background?.equipment) return []
  return props.background.equipment.filter(eq => eq.item?.name !== 'Gold Piece')
})
</script>

<template>
  <UModal
    :open="open"
    @close="emit('close')"
  >
    <template v-if="background">
      <UCard>
        <template #header>
          <div class="flex items-center justify-between">
            <h2 class="text-xl font-bold text-gray-900 dark:text-white">
              {{ background.name }}
            </h2>
            <UButton
              data-test="close-btn"
              icon="i-heroicons-x-mark"
              color="neutral"
              variant="ghost"
              @click="emit('close')"
            />
          </div>
        </template>

        <div class="space-y-6">
          <!-- Feature -->
          <div v-if="background.feature_name">
            <h3 class="font-semibold text-gray-900 dark:text-white mb-2">
              Feature: {{ background.feature_name }}
            </h3>
            <p class="text-gray-600 dark:text-gray-400 text-sm">
              {{ background.feature_description }}
            </p>
          </div>

          <!-- Skill Proficiencies -->
          <div v-if="skillProficiencies.length > 0">
            <h3 class="font-semibold text-gray-900 dark:text-white mb-2">
              Skill Proficiencies
            </h3>
            <div class="flex flex-wrap gap-2">
              <UBadge
                v-for="skill in skillProficiencies"
                :key="skill"
                color="background"
                variant="subtle"
                size="md"
              >
                {{ skill }}
              </UBadge>
            </div>
          </div>

          <!-- Tool Proficiencies -->
          <div v-if="toolProficiencies.length > 0">
            <h3 class="font-semibold text-gray-900 dark:text-white mb-2">
              Tool Proficiencies
            </h3>
            <div class="flex flex-wrap gap-2">
              <UBadge
                v-for="tool in toolProficiencies"
                :key="tool"
                color="background"
                variant="subtle"
                size="md"
              >
                {{ tool }}
              </UBadge>
            </div>
          </div>

          <!-- Languages -->
          <div v-if="languages.length > 0">
            <h3 class="font-semibold text-gray-900 dark:text-white mb-2">
              Languages
            </h3>
            <div class="flex flex-wrap gap-2">
              <UBadge
                v-for="lang in languages"
                :key="lang"
                color="neutral"
                variant="subtle"
                size="md"
              >
                {{ lang }}
              </UBadge>
            </div>
          </div>

          <!-- Equipment -->
          <div v-if="equipmentItems.length > 0 || startingGold">
            <h3 class="font-semibold text-gray-900 dark:text-white mb-2">
              Starting Equipment
            </h3>
            <ul class="list-disc list-inside text-sm text-gray-600 dark:text-gray-400 space-y-1">
              <li
                v-for="eq in equipmentItems"
                :key="eq.id"
              >
                {{ eq.item?.name }}
                <span v-if="eq.quantity > 1">(×{{ eq.quantity }})</span>
                <span
                  v-if="eq.is_choice"
                  class="text-background-500"
                >
                  [choice]
                </span>
              </li>
              <li v-if="startingGold">
                {{ startingGold }} gp
              </li>
            </ul>
          </div>
        </div>
      </UCard>
    </template>
  </UModal>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/BackgroundDetailModal.spec.ts`

Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/BackgroundDetailModal.vue tests/components/character/builder/BackgroundDetailModal.spec.ts
git commit -m "feat(character): add BackgroundDetailModal component"
```

---

## Task 5: StepBackground Component

**Files:**
- Modify: `app/components/character/builder/StepBackground.vue`
- Create: `tests/components/character/builder/StepBackground.spec.ts`

**Step 1: Write the failing test**

Create `tests/components/character/builder/StepBackground.spec.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepBackground from '~/components/character/builder/StepBackground.vue'

const mockBackgrounds = [
  {
    id: 1,
    slug: 'acolyte',
    name: 'Acolyte',
    feature_name: 'Shelter of the Faithful',
    proficiencies: [],
    languages: [],
    equipment: []
  },
  {
    id: 2,
    slug: 'soldier',
    name: 'Soldier',
    feature_name: 'Military Rank',
    proficiencies: [],
    languages: [],
    equipment: []
  }
]

// Mock useApi
vi.mock('~/composables/useApi', () => ({
  useApi: () => ({
    apiFetch: vi.fn().mockResolvedValue({ data: mockBackgrounds })
  })
}))

describe('StepBackground', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('displays search input', async () => {
    const wrapper = await mountSuspended(StepBackground)

    expect(wrapper.find('input[placeholder*="Search"]').exists()).toBe(true)
  })

  it('displays background picker cards', async () => {
    const wrapper = await mountSuspended(StepBackground)

    // Wait for async data
    await wrapper.vm.$nextTick()

    expect(wrapper.text()).toContain('Acolyte')
    expect(wrapper.text()).toContain('Soldier')
  })

  it('filters backgrounds by search query', async () => {
    const wrapper = await mountSuspended(StepBackground)
    await wrapper.vm.$nextTick()

    const input = wrapper.find('input[placeholder*="Search"]')
    await input.setValue('soldier')

    expect(wrapper.text()).toContain('Soldier')
    expect(wrapper.text()).not.toContain('Acolyte')
  })

  it('disables continue button until selection made', async () => {
    const wrapper = await mountSuspended(StepBackground)
    await wrapper.vm.$nextTick()

    const button = wrapper.find('button:contains("Continue")')
    expect(button.attributes('disabled')).toBeDefined()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepBackground.spec.ts`

Expected: FAIL - component is just placeholder

**Step 3: Write minimal implementation**

Replace `app/components/character/builder/StepBackground.vue`:

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import type { Background } from '~/types'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

const store = useCharacterBuilderStore()
const { selectedBackground, isLoading, error } = storeToRefs(store)

// API client
const { apiFetch } = useApi()

// Fetch all backgrounds
const { data: backgrounds, pending: loadingBackgrounds } = await useAsyncData(
  'builder-backgrounds',
  () => apiFetch<{ data: Background[] }>('/backgrounds?per_page=100'),
  { transform: (response: { data: Background[] }) => response.data }
)

// Local state
const searchQuery = ref('')
const localSelectedBackground = ref<Background | null>(null)
const detailModalOpen = ref(false)
const detailBackground = ref<Background | null>(null)

// Apply search filter
const filteredBackgrounds = computed((): Background[] => {
  if (!backgrounds.value) return []
  if (!searchQuery.value) return backgrounds.value
  const query = searchQuery.value.toLowerCase()
  return backgrounds.value.filter((bg: Background) =>
    bg.name.toLowerCase().includes(query) ||
    bg.feature_name?.toLowerCase().includes(query)
  )
})

// Validation: can proceed if background selected
const canProceed = computed(() => {
  return localSelectedBackground.value !== null
})

/**
 * Handle background card selection
 */
function handleBackgroundSelect(background: Background) {
  localSelectedBackground.value = background
}

/**
 * Open detail modal
 */
function handleViewDetails(background: Background) {
  detailBackground.value = background
  detailModalOpen.value = true
}

/**
 * Close detail modal
 */
function handleCloseModal() {
  detailModalOpen.value = false
  detailBackground.value = null
}

/**
 * Confirm selection and call store action
 */
async function confirmSelection() {
  if (!localSelectedBackground.value) return

  try {
    await store.selectBackground(localSelectedBackground.value)
    store.nextStep()
  } catch (err) {
    console.error('Failed to save background:', err)
  }
}

// Initialize from store if already selected
onMounted(() => {
  if (selectedBackground.value) {
    localSelectedBackground.value = selectedBackground.value
  }
})
</script>

<template>
  <div class="space-y-6">
    <!-- Header -->
    <div class="text-center">
      <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
        Choose Your Background
      </h2>
      <p class="mt-2 text-gray-600 dark:text-gray-400">
        Your background reveals where you came from and your place in the world
      </p>
    </div>

    <!-- Search -->
    <div class="max-w-md mx-auto">
      <UInput
        v-model="searchQuery"
        placeholder="Search backgrounds..."
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
      v-if="loadingBackgrounds"
      class="flex justify-center py-12"
    >
      <UIcon
        name="i-heroicons-arrow-path"
        class="w-8 h-8 animate-spin text-background-500"
      />
    </div>

    <!-- Background Grid -->
    <div
      v-else
      class="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 gap-4"
    >
      <CharacterBuilderBackgroundPickerCard
        v-for="background in filteredBackgrounds"
        :key="background.id"
        :background="background"
        :selected="localSelectedBackground?.id === background.id"
        @select="handleBackgroundSelect"
        @view-details="handleViewDetails(background)"
      />
    </div>

    <!-- Empty State -->
    <div
      v-if="!loadingBackgrounds && filteredBackgrounds.length === 0"
      class="text-center py-12"
    >
      <UIcon
        name="i-heroicons-magnifying-glass"
        class="w-12 h-12 text-gray-400 mx-auto mb-4"
      />
      <p class="text-gray-600 dark:text-gray-400">
        No backgrounds found matching "{{ searchQuery }}"
      </p>
    </div>

    <!-- Confirm Button -->
    <div class="flex justify-center pt-4">
      <UButton
        size="lg"
        :disabled="!canProceed || isLoading"
        :loading="isLoading"
        @click="confirmSelection"
      >
        {{ isLoading ? 'Saving...' : 'Continue with ' + (localSelectedBackground?.name || 'Selection') }}
      </UButton>
    </div>

    <!-- Detail Modal -->
    <CharacterBuilderBackgroundDetailModal
      :background="detailBackground"
      :open="detailModalOpen"
      @close="handleCloseModal"
    />
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepBackground.spec.ts`

Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/StepBackground.vue tests/components/character/builder/StepBackground.spec.ts
git commit -m "feat(character): implement StepBackground wizard step"
```

---

## Task 6: EquipmentChoiceGroup Component

**Files:**
- Create: `app/components/character/builder/EquipmentChoiceGroup.vue`
- Create: `tests/components/character/builder/EquipmentChoiceGroup.spec.ts`

**Step 1: Write the failing test**

Create `tests/components/character/builder/EquipmentChoiceGroup.spec.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import EquipmentChoiceGroup from '~/components/character/builder/EquipmentChoiceGroup.vue'

const mockItems = [
  { id: 1, item_id: 101, item: { id: 101, name: 'Longsword' }, quantity: 1, is_choice: true, choice_group: 'weapon', choice_option: 1 },
  { id: 2, item_id: 102, item: { id: 102, name: 'Rapier' }, quantity: 1, is_choice: true, choice_group: 'weapon', choice_option: 2 },
  { id: 3, item_id: 103, item: { id: 103, name: 'Two Shortswords' }, quantity: 2, is_choice: true, choice_group: 'weapon', choice_option: 3 }
]

describe('EquipmentChoiceGroup', () => {
  it('displays group label', async () => {
    const wrapper = await mountSuspended(EquipmentChoiceGroup, {
      props: { groupName: 'Weapon Choice', items: mockItems, selectedItemId: null }
    })

    expect(wrapper.text()).toContain('Weapon Choice')
  })

  it('displays all item options', async () => {
    const wrapper = await mountSuspended(EquipmentChoiceGroup, {
      props: { groupName: 'Weapon Choice', items: mockItems, selectedItemId: null }
    })

    expect(wrapper.text()).toContain('Longsword')
    expect(wrapper.text()).toContain('Rapier')
    expect(wrapper.text()).toContain('Two Shortswords')
  })

  it('shows selected state for chosen item', async () => {
    const wrapper = await mountSuspended(EquipmentChoiceGroup, {
      props: { groupName: 'Weapon Choice', items: mockItems, selectedItemId: 101 }
    })

    const selected = wrapper.find('[data-test="option-101"]')
    expect(selected.classes()).toContain('ring-2')
  })

  it('emits select event when option clicked', async () => {
    const wrapper = await mountSuspended(EquipmentChoiceGroup, {
      props: { groupName: 'Weapon Choice', items: mockItems, selectedItemId: null }
    })

    await wrapper.find('[data-test="option-102"]').trigger('click')

    expect(wrapper.emitted('select')).toBeTruthy()
    expect(wrapper.emitted('select')![0]).toEqual([102])
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/EquipmentChoiceGroup.spec.ts`

Expected: FAIL - component not found

**Step 3: Write minimal implementation**

Create `app/components/character/builder/EquipmentChoiceGroup.vue`:

```vue
<script setup lang="ts">
import type { components } from '~/types/api/generated'

type EntityItemResource = components['schemas']['EntityItemResource']

interface Props {
  groupName: string
  items: EntityItemResource[]
  selectedItemId: number | null
}

defineProps<Props>()

const emit = defineEmits<{
  select: [itemId: number]
}>()

function handleSelect(itemId: number | null) {
  if (itemId) {
    emit('select', itemId)
  }
}
</script>

<template>
  <div class="space-y-2">
    <h4 class="font-medium text-gray-700 dark:text-gray-300">
      {{ groupName }}
    </h4>

    <div class="space-y-2">
      <button
        v-for="item in items"
        :key="item.id"
        :data-test="`option-${item.item_id}`"
        type="button"
        class="w-full p-3 rounded-lg border-2 transition-all text-left flex items-center gap-3"
        :class="[
          selectedItemId === item.item_id
            ? 'ring-2 ring-primary-500 border-primary-500 bg-primary-50 dark:bg-primary-900/20'
            : 'border-gray-200 dark:border-gray-700 hover:border-primary-300'
        ]"
        @click="handleSelect(item.item_id)"
      >
        <!-- Radio indicator -->
        <div
          class="w-5 h-5 rounded-full border-2 flex items-center justify-center flex-shrink-0"
          :class="[
            selectedItemId === item.item_id
              ? 'border-primary-500 bg-primary-500'
              : 'border-gray-400'
          ]"
        >
          <div
            v-if="selectedItemId === item.item_id"
            class="w-2 h-2 rounded-full bg-white"
          />
        </div>

        <!-- Item info -->
        <div>
          <span class="font-medium text-gray-900 dark:text-white">
            {{ item.item?.name }}
          </span>
          <span
            v-if="item.quantity > 1"
            class="text-gray-500 ml-1"
          >
            (×{{ item.quantity }})
          </span>
          <p
            v-if="item.description"
            class="text-sm text-gray-500 dark:text-gray-400"
          >
            {{ item.description }}
          </p>
        </div>
      </button>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/EquipmentChoiceGroup.spec.ts`

Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/EquipmentChoiceGroup.vue tests/components/character/builder/EquipmentChoiceGroup.spec.ts
git commit -m "feat(character): add EquipmentChoiceGroup component"
```

---

## Task 7: Store - Add Equipment State and Actions

**Files:**
- Modify: `app/stores/characterBuilder.ts`
- Test: `tests/stores/characterBuilder.spec.ts`

**Step 1: Write the failing test**

Add to `tests/stores/characterBuilder.spec.ts`:

```typescript
describe('equipment choices', () => {
  it('setEquipmentChoice updates local state', () => {
    const store = useCharacterBuilderStore()

    store.setEquipmentChoice('weapon', 101)

    expect(store.equipmentChoices.get('weapon')).toBe(101)
  })

  it('allEquipmentChoicesMade returns false when choices pending', () => {
    const store = useCharacterBuilderStore()
    store.selectedClass = {
      equipment: [
        { is_choice: true, choice_group: 'weapon', item_id: 101 },
        { is_choice: true, choice_group: 'weapon', item_id: 102 }
      ]
    } as any

    expect(store.allEquipmentChoicesMade).toBe(false)
  })

  it('allEquipmentChoicesMade returns true when all choices made', () => {
    const store = useCharacterBuilderStore()
    store.selectedClass = {
      equipment: [
        { is_choice: true, choice_group: 'weapon', item_id: 101 },
        { is_choice: true, choice_group: 'weapon', item_id: 102 }
      ]
    } as any

    store.setEquipmentChoice('weapon', 101)

    expect(store.allEquipmentChoicesMade).toBe(true)
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.spec.ts -t "equipment choices"`

Expected: FAIL - `setEquipmentChoice is not a function`

**Step 3: Write minimal implementation**

In `app/stores/characterBuilder.ts`, add state:

```typescript
// Equipment choices (choice_group → selected item_id)
const equipmentChoices = ref<Map<string, number>>(new Map())
```

Add computed properties:

```typescript
// Combined equipment from class + background
const allEquipment = computed(() => [
  ...(selectedClass.value?.equipment ?? []),
  ...(selectedBackground.value?.equipment ?? [])
])

// Equipment grouped by choice_group
const equipmentByChoiceGroup = computed(() => {
  const groups = new Map<string, typeof allEquipment.value>()
  for (const item of allEquipment.value) {
    if (item.is_choice && item.choice_group) {
      const existing = groups.get(item.choice_group) ?? []
      groups.set(item.choice_group, [...existing, item])
    }
  }
  return groups
})

// Fixed equipment (no choice required)
const fixedEquipment = computed(() =>
  allEquipment.value.filter(item => !item.is_choice)
)

// Validation: all equipment choices made?
const allEquipmentChoicesMade = computed(() => {
  for (const [group] of equipmentByChoiceGroup.value) {
    if (!equipmentChoices.value.has(group)) return false
  }
  return true
})
```

Add action:

```typescript
/**
 * Set equipment choice (local state only)
 */
function setEquipmentChoice(choiceGroup: string, itemId: number): void {
  equipmentChoices.value.set(choiceGroup, itemId)
}
```

Add to reset function:

```typescript
equipmentChoices.value = new Map()
```

Add to return:

```typescript
equipmentChoices,
allEquipment,
equipmentByChoiceGroup,
fixedEquipment,
allEquipmentChoicesMade,
setEquipmentChoice,
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/stores/characterBuilder.spec.ts -t "equipment choices"`

Expected: PASS

**Step 5: Commit**

```bash
git add app/stores/characterBuilder.ts tests/stores/characterBuilder.spec.ts
git commit -m "feat(character): add equipment choice state and computed properties"
```

---

## Task 8: StepEquipment Component

**Files:**
- Create: `app/components/character/builder/StepEquipment.vue`
- Create: `tests/components/character/builder/StepEquipment.spec.ts`

**Step 1: Write the failing test**

Create `tests/components/character/builder/StepEquipment.spec.ts`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { setActivePinia, createPinia } from 'pinia'
import StepEquipment from '~/components/character/builder/StepEquipment.vue'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

describe('StepEquipment', () => {
  beforeEach(() => {
    setActivePinia(createPinia())
  })

  it('displays class equipment section', async () => {
    const store = useCharacterBuilderStore()
    store.selectedClass = {
      name: 'Fighter',
      equipment: [
        { id: 1, item_id: 1, item: { name: 'Chain Mail' }, quantity: 1, is_choice: false }
      ]
    } as any

    const wrapper = await mountSuspended(StepEquipment)

    expect(wrapper.text()).toContain('Fighter')
    expect(wrapper.text()).toContain('Chain Mail')
  })

  it('displays background equipment section', async () => {
    const store = useCharacterBuilderStore()
    store.selectedClass = { name: 'Fighter', equipment: [] } as any
    store.selectedBackground = {
      name: 'Soldier',
      equipment: [
        { id: 2, item_id: 2, item: { name: 'Insignia of Rank' }, quantity: 1, is_choice: false }
      ]
    } as any

    const wrapper = await mountSuspended(StepEquipment)

    expect(wrapper.text()).toContain('Soldier')
    expect(wrapper.text()).toContain('Insignia of Rank')
  })

  it('displays equipment choice groups', async () => {
    const store = useCharacterBuilderStore()
    store.selectedClass = {
      name: 'Fighter',
      equipment: [
        { id: 1, item_id: 101, item: { name: 'Longsword' }, quantity: 1, is_choice: true, choice_group: 'weapon' },
        { id: 2, item_id: 102, item: { name: 'Rapier' }, quantity: 1, is_choice: true, choice_group: 'weapon' }
      ]
    } as any

    const wrapper = await mountSuspended(StepEquipment)

    expect(wrapper.text()).toContain('Longsword')
    expect(wrapper.text()).toContain('Rapier')
  })

  it('disables continue button until all choices made', async () => {
    const store = useCharacterBuilderStore()
    store.selectedClass = {
      name: 'Fighter',
      equipment: [
        { id: 1, item_id: 101, item: { name: 'Longsword' }, is_choice: true, choice_group: 'weapon' }
      ]
    } as any

    const wrapper = await mountSuspended(StepEquipment)

    const button = wrapper.find('[data-test="continue-btn"]')
    expect(button.attributes('disabled')).toBeDefined()
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepEquipment.spec.ts`

Expected: FAIL - component not found or is placeholder

**Step 3: Write minimal implementation**

Create `app/components/character/builder/StepEquipment.vue`:

```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

const store = useCharacterBuilderStore()
const {
  selectedClass,
  selectedBackground,
  fixedEquipment,
  equipmentByChoiceGroup,
  equipmentChoices,
  allEquipmentChoicesMade,
  isLoading
} = storeToRefs(store)

// Separate equipment by source
const classFixedEquipment = computed(() =>
  selectedClass.value?.equipment?.filter(eq => !eq.is_choice) ?? []
)

const backgroundFixedEquipment = computed(() =>
  selectedBackground.value?.equipment?.filter(eq => !eq.is_choice) ?? []
)

// Get choice groups by source
const classChoiceGroups = computed(() => {
  const groups = new Map()
  for (const item of selectedClass.value?.equipment ?? []) {
    if (item.is_choice && item.choice_group) {
      const existing = groups.get(item.choice_group) ?? []
      groups.set(item.choice_group, [...existing, item])
    }
  }
  return groups
})

const backgroundChoiceGroups = computed(() => {
  const groups = new Map()
  for (const item of selectedBackground.value?.equipment ?? []) {
    if (item.is_choice && item.choice_group) {
      const existing = groups.get(item.choice_group) ?? []
      groups.set(item.choice_group, [...existing, item])
    }
  }
  return groups
})

/**
 * Handle equipment choice selection
 */
function handleChoiceSelect(choiceGroup: string, itemId: number) {
  store.setEquipmentChoice(choiceGroup, itemId)
}

/**
 * Format choice group name for display
 */
function formatGroupName(group: string): string {
  return group
    .replace(/-/g, ' ')
    .replace(/\b\w/g, l => l.toUpperCase())
}

/**
 * Continue to next step
 */
function handleContinue() {
  store.nextStep()
}
</script>

<template>
  <div class="space-y-6">
    <!-- Header -->
    <div class="text-center">
      <h2 class="text-2xl font-bold text-gray-900 dark:text-white">
        Choose Your Starting Equipment
      </h2>
      <p class="mt-2 text-gray-600 dark:text-gray-400">
        Select your starting gear from your class and background
      </p>
    </div>

    <!-- Class Equipment -->
    <div
      v-if="selectedClass"
      class="space-y-4"
    >
      <h3 class="text-lg font-semibold text-gray-900 dark:text-white border-b pb-2">
        From Your Class ({{ selectedClass.name }})
      </h3>

      <!-- Fixed Items -->
      <div
        v-if="classFixedEquipment.length > 0"
        class="space-y-2"
      >
        <div
          v-for="item in classFixedEquipment"
          :key="item.id"
          class="flex items-center gap-2 text-gray-700 dark:text-gray-300"
        >
          <UIcon
            name="i-heroicons-check-circle"
            class="w-5 h-5 text-green-500"
          />
          <span>{{ item.item?.name }}</span>
          <span
            v-if="item.quantity > 1"
            class="text-gray-500"
          >(×{{ item.quantity }})</span>
        </div>
      </div>

      <!-- Choice Groups -->
      <CharacterBuilderEquipmentChoiceGroup
        v-for="[group, items] in classChoiceGroups"
        :key="group"
        :group-name="formatGroupName(group)"
        :items="items"
        :selected-item-id="equipmentChoices.get(group) ?? null"
        @select="(itemId) => handleChoiceSelect(group, itemId)"
      />
    </div>

    <!-- Background Equipment -->
    <div
      v-if="selectedBackground"
      class="space-y-4"
    >
      <h3 class="text-lg font-semibold text-gray-900 dark:text-white border-b pb-2">
        From Your Background ({{ selectedBackground.name }})
      </h3>

      <!-- Fixed Items -->
      <div
        v-if="backgroundFixedEquipment.length > 0"
        class="space-y-2"
      >
        <div
          v-for="item in backgroundFixedEquipment"
          :key="item.id"
          class="flex items-center gap-2 text-gray-700 dark:text-gray-300"
        >
          <UIcon
            name="i-heroicons-check-circle"
            class="w-5 h-5 text-green-500"
          />
          <span>{{ item.item?.name }}</span>
          <span
            v-if="item.quantity > 1"
            class="text-gray-500"
          >(×{{ item.quantity }})</span>
        </div>
      </div>

      <!-- Choice Groups -->
      <CharacterBuilderEquipmentChoiceGroup
        v-for="[group, items] in backgroundChoiceGroups"
        :key="group"
        :group-name="formatGroupName(group)"
        :items="items"
        :selected-item-id="equipmentChoices.get(group) ?? null"
        @select="(itemId) => handleChoiceSelect(group, itemId)"
      />
    </div>

    <!-- Continue Button -->
    <div class="flex justify-center pt-4">
      <UButton
        data-test="continue-btn"
        size="lg"
        :disabled="!allEquipmentChoicesMade || isLoading"
        :loading="isLoading"
        @click="handleContinue"
      >
        Continue with Equipment
      </UButton>
    </div>
  </div>
</template>
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/character/builder/StepEquipment.spec.ts`

Expected: PASS

**Step 5: Commit**

```bash
git add app/components/character/builder/StepEquipment.vue tests/components/character/builder/StepEquipment.spec.ts
git commit -m "feat(character): implement StepEquipment wizard step"
```

---

## Remaining Tasks (Summary)

The following tasks follow the same TDD pattern:

### Task 9: SpellPickerCard Component
- Compact selectable spell card
- Shows: name, level, school, concentration badge
- Checkbox/toggle selection

### Task 10: Store - Add Spell Actions
- `learnSpell(spellId)` - POST to API
- `unlearnSpell(spellId)` - DELETE from API
- `setRaceSpellChoice(group, spellId)` - local state

### Task 11: StepSpells Component
- Race spell section (fixed + choices)
- Class cantrips section
- Class spells section
- Validation against limits

### Task 12: StepReview Component
- Summary sections for all data
- Final "Create Character" button
- Validation check

### Task 13: Update Wizard Page
- Update step rendering logic
- Handle 7 vs 8 step flow
- Update stepper component

### Task 14: Integration Tests
- Full wizard flow test
- Equipment choice persistence
- Spell selection limits

---

## Execution Notes

- Run domain test suite after each task: `npm run test:classes`
- Run full suite before PR: `npm run test`
- Run typecheck: `npm run typecheck`
- Run lint: `npm run lint:fix`

---

**Plan complete and saved to `docs/plans/2025-12-03-character-builder-phase-4-implementation.md`.**

Two execution options:

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

Which approach?
