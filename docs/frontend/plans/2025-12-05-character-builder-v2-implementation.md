# Character Builder v2 - Implementation Plan

**Date:** 2025-12-05
**Design Doc:** `docs/plans/2025-12-05-character-builder-v2-design.md`
**Branch:** `feature/character-builder-v2`
**Worktree:** `/Users/dfox/Development/dnd/frontend-agent-2`

---

## Task Overview

| Wave | Tasks | Parallel? | Estimated Scope |
|------|-------|-----------|-----------------|
| 1 | Foundation (store, composables, types) | No | 4 tasks |
| 2 | Core Components (layout, stats cards) | Yes | 6 tasks |
| 3 | Step Components (reuse + new) | Yes | 5 tasks |
| 4 | Integration & Polish | No | 3 tasks |

---

## Wave 1: Foundation (Sequential)

These must be done first as other tasks depend on them.

### Task 1.1: Create Lean Store

**File:** `app/stores/characterWizard.ts`

**Implementation:**
```typescript
import { defineStore } from 'pinia'

interface Selections {
  race: Race | null
  subrace: Race | null
  class: CharacterClass | null
  subclass: Subclass | null
  background: Background | null
  abilityScores: AbilityScores
  abilityMethod: 'standard_array' | 'point_buy' | 'manual'
  name: string
  alignment: Alignment | null
}

interface PendingChoices {
  proficiencies: Map<string, Set<number>>
  languages: Map<string, Set<number>>
  equipment: Map<string, number>
  spells: Set<number>
}

export const useCharacterWizardStore = defineStore('characterWizard', () => {
  // Character identity
  const characterId = ref<number | null>(null)
  const selectedSources = ref<string[]>([])

  // Selections
  const selections = ref<Selections>({ /* defaults */ })

  // Pending choices
  const pendingChoices = ref<PendingChoices>({ /* defaults */ })

  // Backend data
  const stats = ref<CharacterStats | null>(null)
  const summary = ref<CharacterSummary | null>(null)

  // UI state
  const isLoading = ref(false)
  const error = ref<string | null>(null)

  // Computed: step visibility
  const needsSubraceStep = computed(() => { /* ... */ })
  const needsSubclassStep = computed(() => { /* ... */ })
  const isSpellcaster = computed(() => { /* ... */ })
  const hasProficiencyChoices = computed(() => { /* ... */ })
  const hasLanguageChoices = computed(() => { /* ... */ })

  // Actions
  async function syncStats() { /* fetch /stats and /summary */ }
  function reset() { /* clear all state */ }

  return { /* all refs and functions */ }
})
```

**Test:** `tests/stores/characterWizard.test.ts`
- Test initial state
- Test computed step visibility
- Test reset function

---

### Task 1.2: Create useCharacterWizard Composable

**File:** `app/composables/useCharacterWizard.ts`

**Implementation:**
```typescript
export interface WizardStep {
  name: string
  label: string
  icon: string
  visible: () => boolean
}

const stepRegistry: WizardStep[] = [
  { name: 'sourcebooks', label: 'Sources', icon: 'i-heroicons-book-open', visible: () => true },
  { name: 'race', label: 'Race', icon: 'i-heroicons-globe-alt', visible: () => true },
  { name: 'subrace', label: 'Subrace', icon: 'i-heroicons-sparkles', visible: () => store.needsSubraceStep },
  { name: 'class', label: 'Class', icon: 'i-heroicons-shield-check', visible: () => true },
  { name: 'subclass', label: 'Subclass', icon: 'i-heroicons-star', visible: () => store.needsSubclassStep },
  { name: 'background', label: 'Background', icon: 'i-heroicons-book-open', visible: () => true },
  { name: 'abilities', label: 'Abilities', icon: 'i-heroicons-chart-bar', visible: () => true },
  { name: 'proficiencies', label: 'Skills', icon: 'i-heroicons-academic-cap', visible: () => store.hasProficiencyChoices },
  { name: 'languages', label: 'Languages', icon: 'i-heroicons-language', visible: () => store.hasLanguageChoices },
  { name: 'equipment', label: 'Equipment', icon: 'i-heroicons-briefcase', visible: () => true },
  { name: 'spells', label: 'Spells', icon: 'i-heroicons-sparkles', visible: () => store.isSpellcaster },
  { name: 'details', label: 'Details', icon: 'i-heroicons-user', visible: () => true },
  { name: 'review', label: 'Review', icon: 'i-heroicons-check-circle', visible: () => true },
]

export function useCharacterWizard() {
  const store = useCharacterWizardStore()
  const route = useRoute()

  const activeSteps = computed(() => stepRegistry.filter(s => s.visible()))
  const currentStepName = computed(() => extractStepFromPath(route.path))
  const currentStepIndex = computed(() => activeSteps.value.findIndex(s => s.name === currentStepName.value))
  const canProceed = computed(() => validateCurrentStep())

  async function nextStep() { /* navigate */ }
  async function previousStep() { /* navigate */ }
  async function saveAndContinue() { /* save current, then next */ }

  return { activeSteps, currentStepName, currentStepIndex, canProceed, nextStep, previousStep, saveAndContinue }
}
```

**Test:** `tests/composables/useCharacterWizard.test.ts`
- Test step filtering
- Test navigation logic

---

### Task 1.3: Create useCharacterStats Composable

**File:** `app/composables/useCharacterStats.ts`

**Implementation:**
```typescript
export function useCharacterStats(characterId: Ref<number | null>) {
  const { apiFetch } = useApi()
  const stats = ref<CharacterStats | null>(null)
  const isLoading = ref(false)

  // Formatted display values
  const hitPoints = computed(() => stats.value?.hit_points?.max ?? '—')
  const armorClass = computed(() => stats.value?.armor_class ?? 10)
  const initiative = computed(() => formatModifier(stats.value?.initiative_bonus))
  const proficiencyBonus = computed(() => `+${stats.value?.proficiency_bonus ?? 2}`)
  const passivePerception = computed(() => stats.value?.passive_perception ?? 10)

  const abilityScores = computed(() => {
    if (!stats.value?.ability_scores) return null
    return Object.entries(stats.value.ability_scores).map(([code, data]) => ({
      code,
      score: data.score,
      modifier: data.modifier,
      formatted: `${data.score} (${formatModifier(data.modifier)})`
    }))
  })

  const savingThrows = computed(() => {
    if (!stats.value?.saving_throws) return null
    // Format with proficiency indicators
  })

  const spellcasting = computed(() => stats.value?.spellcasting ?? null)

  async function refresh() {
    if (!characterId.value) return
    isLoading.value = true
    try {
      const res = await apiFetch(`/characters/${characterId.value}/stats`)
      stats.value = res.data
    } finally {
      isLoading.value = false
    }
  }

  return { stats, isLoading, hitPoints, armorClass, initiative, proficiencyBonus, abilityScores, savingThrows, spellcasting, refresh }
}

function formatModifier(mod: number | null | undefined): string {
  if (mod == null) return '—'
  return mod >= 0 ? `+${mod}` : `${mod}`
}
```

**Test:** `tests/composables/useCharacterStats.test.ts`
- Test formatting functions
- Test computed values with mock data

---

### Task 1.4: Create Nitro Routes for New Endpoints

**Files:**
- `server/api/characters/[id]/summary.get.ts`
- `server/api/classes/[slug]/subclasses.get.ts` (if needed)

**Implementation:**
```typescript
// server/api/characters/[id]/summary.get.ts
export default defineEventHandler(async (event) => {
  const config = useRuntimeConfig()
  const id = getRouterParam(event, 'id')
  const data = await $fetch(`${config.apiBaseServer}/characters/${id}/summary`)
  return data
})
```

**Test:** Manual verification via curl

---

## Wave 2: Core Components (Parallel)

These can be built in parallel by multiple agents.

### Task 2.1: WizardLayout Component

**File:** `app/components/character/wizard/WizardLayout.vue`

**Implementation:**
```vue
<template>
  <div class="min-h-screen flex">
    <!-- Sidebar -->
    <WizardSidebar class="w-64 flex-shrink-0" />

    <!-- Main content -->
    <div class="flex-1 flex flex-col">
      <main class="flex-1 p-6 overflow-y-auto">
        <slot />
      </main>

      <!-- Footer -->
      <WizardFooter />
    </div>
  </div>
</template>
```

**Test:** `tests/components/character/wizard/WizardLayout.test.ts`

---

### Task 2.2: WizardSidebar Component

**File:** `app/components/character/wizard/WizardSidebar.vue`

**Implementation:**
- Show all steps with icons
- Current step highlighted
- Completed steps with checkmark
- Hidden steps not shown
- Click to navigate (if allowed)

**Test:** `tests/components/character/wizard/WizardSidebar.test.ts`

---

### Task 2.3: WizardFooter Component

**File:** `app/components/character/wizard/WizardFooter.vue`

**Implementation:**
- Back button (disabled on first step)
- Next/Continue button (disabled if validation fails)
- "Finish" on review step

**Test:** `tests/components/character/wizard/WizardFooter.test.ts`

---

### Task 2.4: CombatStatsCard Component

**File:** `app/components/character/stats/CombatStatsCard.vue`

**Props:**
```typescript
interface Props {
  hitPoints: number | string
  armorClass: number
  initiative: string
  speed: number
  proficiencyBonus: string
}
```

**Test:** `tests/components/character/stats/CombatStatsCard.test.ts`

---

### Task 2.5: SavingThrowsCard Component

**File:** `app/components/character/stats/SavingThrowsCard.vue`

**Props:**
```typescript
interface Props {
  savingThrows: Array<{
    code: string
    bonus: number
    isProficient: boolean
  }>
}
```

**Display:** Grid of 6 saves with proficiency dots (●)

**Test:** `tests/components/character/stats/SavingThrowsCard.test.ts`

---

### Task 2.6: SpellcastingCard Component

**File:** `app/components/character/stats/SpellcastingCard.vue`

**Props:**
```typescript
interface Props {
  ability: string
  saveDC: number
  attackBonus: number
  slots: Array<{ level: number, total: number }>
}
```

**Test:** `tests/components/character/stats/SpellcastingCard.test.ts`

---

## Wave 3: Step Components (Parallel)

### Task 3.1: StepSubclass Component (NEW)

**File:** `app/components/character/wizard/StepSubclass.vue`

**Implementation:**
- Fetch subclasses for selected class
- Filter by selected sourcebooks
- Grid of SubclassPickerCard components
- Show subclass features on selection

**Dependencies:**
- Create `SubclassPickerCard.vue`
- Create `SubclassDetailModal.vue`

**Test:** `tests/components/character/wizard/StepSubclass.test.ts`
- Shows for Cleric, Sorcerer, Warlock
- Filters by sourcebook
- Emits selection

---

### Task 3.2: StepDetails Component (NEW)

**File:** `app/components/character/wizard/StepDetails.vue`

**Implementation:**
- Name input field
- Alignment dropdown with tooltips
- Moved from old StepName, positioned at end

**Test:** `tests/components/character/wizard/StepDetails.test.ts`

---

### Task 3.3: StepReview Component (REWRITE)

**File:** `app/components/character/wizard/StepReview.vue`

**Implementation:**
- Use all stats card components
- Show complete character sheet
- Edit buttons to jump to steps
- "Finish" button

**Test:** `tests/components/character/wizard/StepReview.test.ts`

---

### Task 3.4: Rewire Existing Steps

Migrate existing step components to use new store/composables:
- `StepSourcebooks.vue` - minimal changes
- `StepRace.vue` - update store imports
- `StepSubrace.vue` - add "None" option for optional subraces
- `StepClass.vue` - update store imports
- `StepBackground.vue` - update store imports
- `StepAbilities.vue` - show modifiers, update store
- `StepProficiencies.vue` - update store imports
- `StepLanguages.vue` - update store imports
- `StepEquipment.vue` - update store imports
- `StepSpells.vue` - add spellcasting info display

**Test:** Update existing tests to use new store

---

### Task 3.5: SubclassPickerCard Component (NEW)

**File:** `app/components/character/picker/SubclassPickerCard.vue`

**Implementation:**
- Similar pattern to ClassPickerCard
- Show subclass name, source
- Show key features preview

**Test:** `tests/components/character/picker/SubclassPickerCard.test.ts`

---

## Wave 4: Integration & Polish (Sequential)

### Task 4.1: Page Routes

**Files:**
- `app/pages/characters/new/index.vue` - redirect to sourcebooks
- `app/pages/characters/new/[step].vue` - dynamic step router

**Implementation:**
```vue
<!-- app/pages/characters/new/[step].vue -->
<template>
  <CharacterWizardLayout>
    <component :is="stepComponent" />
  </CharacterWizardLayout>
</template>

<script setup lang="ts">
const stepComponents: Record<string, Component> = {
  sourcebooks: resolveComponent('CharacterWizardStepSourcebooks'),
  race: resolveComponent('CharacterWizardStepRace'),
  // ... all steps
}

const { currentStepName } = useCharacterWizard()
const stepComponent = computed(() => stepComponents[currentStepName.value])
</script>
```

---

### Task 4.2: Integration Testing

**File:** `tests/pages/characters/new/wizard-flow.test.ts`

**Test Cases:**
- Full wizard flow: Sourcebooks → Race → Class → Background → Abilities → Equipment → Details → Review
- Conditional steps appear/hide correctly
- Character is created on race selection
- Stats update after each save
- Can complete and view character

---

### Task 4.3: Cleanup & Migration

- Update `app/pages/characters/create.vue` to redirect to `/characters/new`
- Mark old store as deprecated (don't delete yet)
- Update any imports in existing character view page
- Remove unused code after verification

---

## Parallelization Strategy

```
┌─────────────────────────────────────────────────────────────┐
│                      WAVE 1 (Sequential)                    │
│  Task 1.1 → Task 1.2 → Task 1.3 → Task 1.4                 │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    WAVE 2 (Parallel - 3 agents)             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Agent A     │  │ Agent B     │  │ Agent C     │         │
│  │ 2.1 Layout  │  │ 2.4 Combat  │  │ 2.2 Sidebar │         │
│  │ 2.3 Footer  │  │ 2.5 Saves   │  │ 2.6 Spells  │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                    WAVE 3 (Parallel - 3 agents)             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐         │
│  │ Agent A     │  │ Agent B     │  │ Agent C     │         │
│  │ 3.1 Subclass│  │ 3.2 Details │  │ 3.4 Rewire  │         │
│  │ 3.5 Picker  │  │ 3.3 Review  │  │    Steps    │         │
│  └─────────────┘  └─────────────┘  └─────────────┘         │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                      WAVE 4 (Sequential)                    │
│  Task 4.1 → Task 4.2 → Task 4.3                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Definition of Done

- [ ] All tests pass (`npm run test`)
- [ ] TypeScript compiles (`npm run typecheck`)
- [ ] ESLint passes (`npm run lint`)
- [ ] Manual testing: create Cleric with subclass, verify stats
- [ ] Manual testing: create Fighter (no subclass), verify flow
- [ ] Manual testing: Human without subrace works
- [ ] Review step shows all calculated stats
- [ ] GitHub issues can be closed: #175, #176, #177, #178, #179, #180, #181, #184
