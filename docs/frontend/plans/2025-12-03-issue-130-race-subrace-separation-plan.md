# Issue 130: Race/Subrace Step Separation - Implementation Plan

## Overview

Split race and subrace selection into two distinct wizard steps. This plan follows TDD - tests are written before implementation.

## Task Breakdown

### Task 1: Update Store - Add hasSubraces and clearSubrace

**File:** `app/stores/characterBuilder.ts`

**Changes:**

1. Add `hasSubraces` computed property:
```typescript
const hasSubraces = computed(() =>
  (selectedRace.value?.subraces?.length ?? 0) > 0
)
```

2. Update `totalSteps` to include conditional subrace step:
```typescript
const totalSteps = computed(() => {
  let steps = 7 // Base: Name, Race, Class, Abilities, Background, Equipment, Review
  if (hasSubraces.value) steps++ // Subrace step
  if (hasPendingChoices.value) steps++
  if (isCaster.value) steps++
  return steps
})
```

3. Add `clearSubrace` action:
```typescript
function clearSubrace(): void {
  subraceId.value = null
  // If selectedRace is a subrace, reset to parent
  if (selectedRace.value?.parent_race) {
    // We need to refetch the parent race or keep it cached
  }
}
```

4. Modify `selectRace` to only save base race (subrace saved separately):
```typescript
async function selectRace(race: Race): Promise<void> {
  // Save only base race to API
  await apiFetch(`/characters/${characterId.value}`, {
    method: 'PATCH',
    body: { race_id: race.id }
  })

  raceId.value = race.id
  subraceId.value = null
  selectedRace.value = race

  await refreshStats()
}
```

5. Add new `selectSubrace` action:
```typescript
async function selectSubrace(subrace: Race): Promise<void> {
  // Save subrace to API (overwrites race_id with subrace id)
  await apiFetch(`/characters/${characterId.value}`, {
    method: 'PATCH',
    body: { race_id: subrace.id }
  })

  subraceId.value = subrace.id
  selectedRace.value = subrace

  await refreshStats()
}
```

6. Export new properties/actions:
- `hasSubraces`
- `clearSubrace`
- `selectSubrace`

**Test File:** `tests/stores/characterBuilder.test.ts`

Add tests:
- `hasSubraces returns false when selectedRace has no subraces`
- `hasSubraces returns true when selectedRace has subraces`
- `clearSubrace clears subraceId`
- `selectSubrace saves subrace to API`
- `totalSteps includes subrace step when hasSubraces is true`

---

### Task 2: Modify StepRace.vue - Remove Subrace Selection

**File:** `app/components/character/builder/StepRace.vue`

**Changes:**

1. Remove subrace-related state:
   - Remove `localSelectedSubrace`
   - Remove `selectedRaceHasSubraces` computed
   - Remove `availableSubraces` computed

2. Remove subrace-related functions:
   - Remove `handleSubraceSelect`

3. Simplify `canProceed`:
```typescript
const canProceed = computed(() => !!localSelectedRace.value)
```

4. Modify `confirmSelection` to only save race:
```typescript
async function confirmSelection() {
  if (!localSelectedRace.value) return

  try {
    await store.selectRace(localSelectedRace.value)
    store.nextStep()
  } catch (err) {
    console.error('Failed to save race:', err)
  }
}
```

5. Add confirmation modal for changing race when subrace exists:
```vue
<UModal v-model:open="confirmChangeModalOpen">
  <template #header>Change Race?</template>
  <template #body>
    Changing race will clear your subrace selection. Continue?
  </template>
  <template #footer>
    <UButton variant="outline" @click="confirmChangeModalOpen = false">Cancel</UButton>
    <UButton color="primary" @click="confirmRaceChange">Continue</UButton>
  </template>
</UModal>
```

6. Add logic for confirmation:
```typescript
const confirmChangeModalOpen = ref(false)
const pendingRaceChange = ref<Race | null>(null)

function handleRaceSelect(race: Race) {
  // If changing race and subrace exists, show confirmation
  if (store.subraceId && localSelectedRace.value?.id !== race.id) {
    pendingRaceChange.value = race
    confirmChangeModalOpen.value = true
    return
  }
  localSelectedRace.value = race
}

function confirmRaceChange() {
  if (pendingRaceChange.value) {
    store.clearSubrace()
    localSelectedRace.value = pendingRaceChange.value
    pendingRaceChange.value = null
  }
  confirmChangeModalOpen.value = false
}
```

7. Remove subrace selector template section (lines 193-234)

8. Update button text:
```vue
{{ isLoading ? 'Saving...' : 'Continue with ' + (localSelectedRace?.name || 'Selection') }}
```

**Test Updates:** `tests/components/character/builder/StepRace.test.ts`
- Update existing tests to not expect subrace selection
- Add test for confirmation modal when changing race with existing subrace

---

### Task 3: Create StepSubrace.vue Component

**File:** `app/components/character/builder/StepSubrace.vue`

**Template Structure:**
```vue
<script setup lang="ts">
import { storeToRefs } from 'pinia'
import type { Race } from '~/types'
import { useCharacterBuilderStore } from '~/stores/characterBuilder'

const store = useCharacterBuilderStore()
const { selectedRace, subraceId, isLoading, error } = storeToRefs(store)
const { apiFetch } = useApi()

// Fetch all races to get full subrace data
const { data: races } = await useAsyncData(
  'builder-races-subraces',
  () => apiFetch<{ data: Race[] }>('/races?per_page=100'),
  { transform: (response) => response.data }
)

// Local state
const localSelectedSubrace = ref<Race | null>(null)
const detailModalOpen = ref(false)
const detailSubrace = ref<Race | null>(null)

// Get full subrace objects for selected race
const availableSubraces = computed((): Race[] => {
  if (!selectedRace.value || !races.value) return []
  return races.value.filter((race: Race) =>
    race.parent_race?.id === selectedRace.value?.id
  )
})

const canProceed = computed(() => !!localSelectedSubrace.value)

function handleSubraceSelect(subrace: Race) {
  localSelectedSubrace.value = subrace
}

function handleViewDetails(subrace: Race) {
  detailSubrace.value = subrace
  detailModalOpen.value = true
}

async function confirmSelection() {
  if (!localSelectedSubrace.value) return

  try {
    await store.selectSubrace(localSelectedSubrace.value)
    store.nextStep()
  } catch (err) {
    console.error('Failed to save subrace:', err)
  }
}

// Initialize from store if already selected
onMounted(() => {
  if (subraceId.value && races.value) {
    localSelectedSubrace.value = races.value.find(r => r.id === subraceId.value) ?? null
  }
})
</script>
```

**Template:**
- Header: "Choose Your Subrace" with parent race name
- Grid of subrace cards (reuse RacePickerCard or create SubraceCard)
- Each card shows: name, ability modifiers, traits preview
- Detail modal for expanded view
- Continue button

**Test File:** `tests/components/character/builder/StepSubrace.test.ts`
- Renders available subraces for selected race
- Selecting subrace enables continue button
- Confirm saves subrace and advances step
- Initializes from store when subrace already selected
- Detail modal opens on view details click

---

### Task 4: Update edit.vue - Add Conditional Subrace Step

**File:** `app/pages/characters/[id]/edit.vue`

**Changes:**

1. Add `hasSubraces` to storeToRefs:
```typescript
const { currentStep, isFirstStep, isLastStep, isCaster, hasPendingChoices, hasSubraces, isLoading, error, name } = storeToRefs(store)
```

2. Update steps computed to include conditional subrace step:
```typescript
const steps = computed(() => {
  const stepList = [
    { id: 1, name: 'name', label: 'Name', icon: 'i-heroicons-user' },
    { id: 2, name: 'race', label: 'Race', icon: 'i-heroicons-globe-alt' }
  ]

  let nextId = 3

  // Conditional subrace step (after race, only if race has subraces)
  if (hasSubraces.value) {
    stepList.push({ id: nextId++, name: 'subrace', label: 'Subrace', icon: 'i-heroicons-globe-americas' })
  }

  stepList.push(
    { id: nextId++, name: 'class', label: 'Class', icon: 'i-heroicons-shield-check' },
    { id: nextId++, name: 'abilities', label: 'Abilities', icon: 'i-heroicons-chart-bar' },
    { id: nextId++, name: 'background', label: 'Background', icon: 'i-heroicons-book-open' }
  )

  // Conditional proficiency choices step
  if (hasPendingChoices.value) {
    stepList.push({ id: nextId++, name: 'proficiencies', label: 'Proficiencies', icon: 'i-heroicons-academic-cap' })
  }

  stepList.push({ id: nextId++, name: 'equipment', label: 'Equipment', icon: 'i-heroicons-briefcase' })

  // Conditional spells step
  if (isCaster.value) {
    stepList.push({ id: nextId++, name: 'spells', label: 'Spells', icon: 'i-heroicons-sparkles' })
  }

  stepList.push({ id: nextId, name: 'review', label: 'Review', icon: 'i-heroicons-check-circle' })

  return stepList
})
```

3. Add Suspense block for subrace step in template:
```vue
<Suspense v-else-if="currentStepName === 'subrace'">
  <CharacterBuilderStepSubrace />
  <template #fallback>
    <div class="flex justify-center py-12">
      <UIcon name="i-heroicons-arrow-path" class="w-8 h-8 animate-spin text-primary" />
    </div>
  </template>
</Suspense>
```

---

### Task 5: Update Store - Edit Mode Subrace Handling

**File:** `app/stores/characterBuilder.ts`

**Changes to `loadCharacterForEditing`:**

Currently the code detects subraces and sets both `raceId` and `subraceId`. We need to also load the parent race into state for proper `hasSubraces` calculation:

```typescript
if (character.race) {
  const raceResponse = await apiFetch<{ data: Race }>(`/races/${character.race.slug}`)
  const loadedRace = raceResponse.data

  if (loadedRace.parent_race) {
    // This is a subrace - fetch parent race for hasSubraces to work
    const parentResponse = await apiFetch<{ data: Race }>(`/races/${loadedRace.parent_race.slug}`)
    selectedRace.value = parentResponse.data  // Store parent race (has subraces array)
    subraceId.value = loadedRace.id
    raceId.value = loadedRace.parent_race.id
  } else {
    // Base race - store as-is
    selectedRace.value = loadedRace
    raceId.value = loadedRace.id
    subraceId.value = null
  }
}
```

**Changes to `determineStartingStep`:**

Account for subrace step in step calculation:
```typescript
function determineStartingStep(character: Character): number {
  if (!character.name) return 1
  if (!character.race) return 2
  // If race has subraces but no subrace selected, start at subrace step
  if (selectedRace.value?.subraces?.length && !subraceId.value) return 3
  if (!character.class) return hasSubraces.value ? 4 : 3
  if (!character.ability_scores?.STR) return hasSubraces.value ? 5 : 4
  if (!character.background) return hasSubraces.value ? 6 : 5
  return hasSubraces.value ? 7 : 6
}
```

---

### Task 6: Tests for All Changes

**Files to create/update:**
1. `tests/stores/characterBuilder.test.ts` - Store tests
2. `tests/components/character/builder/StepRace.test.ts` - Updated tests
3. `tests/components/character/builder/StepSubrace.test.ts` - New test file

---

## Implementation Order

1. **Task 1:** Store changes (hasSubraces, clearSubrace, selectSubrace, totalSteps)
2. **Task 3:** Create StepSubrace.vue component
3. **Task 2:** Modify StepRace.vue (remove subrace UI, add confirmation modal)
4. **Task 4:** Update edit.vue (add conditional step)
5. **Task 5:** Edit mode handling improvements
6. **Task 6:** Comprehensive test coverage

## Verification Steps

1. Run `npm run test:core` for store tests
2. Run `npm run test:ui` for component tests
3. Manual browser testing:
   - Create character with race that has subraces (e.g., Elf)
   - Verify Race step shows only race selection
   - Verify Subrace step appears after race selection
   - Verify navigation back and forth works
   - Verify changing race shows confirmation if subrace selected
   - Create character with race without subraces (e.g., Human)
   - Verify Subrace step does NOT appear
4. Run full test suite: `npm run test`
5. Run typecheck: `npm run typecheck`
6. Run lint: `npm run lint:fix`
