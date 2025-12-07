# Character Builder Frontend - Design Document

**Date:** 2025-12-01
**Status:** Approved for Implementation
**Backend Issue:** #21 (Complete)
**Related:** `importer/docs/plans/2025-11-30-character-builder-api-design.md`

---

## Overview

### Goal

Build a guided wizard for creating D&D 5e characters using the Character Builder API (v1). The wizard walks players through character creation step-by-step while showing real-time stat calculations.

### User Outcome

- New players can create valid D&D 5e characters without rulebook knowledge
- Experienced players get a streamlined creation flow
- Real-time feedback shows how choices affect stats
- Mobile-friendly interface for table-side character creation

### Scope

**In Scope (v1):**
- Multi-step creation wizard (7 steps)
- Character list view
- Character sheet view (read-only)
- Manual ability score entry (3-20 range)
- Spell selection for spellcasting classes

**Out of Scope (v2+):**
- Point buy / Standard array (#87)
- Multiclass support
- Level up flow
- Equipment management
- User authentication / character ownership
- Character editing after creation

---

## Architecture Decision

### Approach: Single-Page Wizard with Stepper

After evaluating three approaches (single-page wizard, multi-page routes, drawer/modal), we selected the **single-page wizard** because:

1. **Smooth UX** - No page reloads between steps
2. **Real-time preview** - Sidebar shows stat changes as user makes choices
3. **NuxtUI alignment** - Leverages `UStepper` component
4. **Mobile-friendly** - Single scrollable page works well on phones
5. **Pattern consistency** - Similar to existing tabbed views (class detail page)

### URL Structure

```
/characters              → Character list
/characters/create       → Creation wizard (single page)
/characters/[id]         → Character sheet (view mode)
```

---

## Wizard Flow

### Step Sequence

```
① Name ──▶ ② Race ──▶ ③ Class ──▶ ④ Abilities ──▶ ⑤ Background ──▶ ⑥ Spells* ──▶ ⑦ Review
                                                                      │
                                                        *Only if class has spellcasting
```

### Step Details

| Step | Name | Required Fields | API Action | Validation |
|------|------|-----------------|------------|------------|
| 1 | Name | `name` | `POST /characters` | Non-empty, max 255 chars |
| 2 | Race | `race_id`, `subrace_id`? | `PATCH /characters/{id}` | Valid race, subrace if required |
| 3 | Class | `class_id` | `PATCH /characters/{id}` | Valid class |
| 4 | Abilities | 6 scores | `PATCH /characters/{id}` | Each score 3-20 |
| 5 | Background | `background_id` | `PATCH /characters/{id}` | Valid background |
| 6 | Spells | spell selections | `POST /characters/{id}/spells` | Valid spells for class/level |
| 7 | Review | — | `GET /characters/{id}/stats` | All fields complete |

### Step Order Rationale

1. **Name first** - Creates the character draft, establishes identity
2. **Race before Class** - Racial bonuses inform class selection (e.g., +2 DEX suggests Rogue)
3. **Class before Abilities** - Players need to know which abilities matter for their class
4. **Background last** - Mostly flavor, doesn't affect other mechanical choices
5. **Spells conditional** - Only shown for classes with `spellcasting_ability !== null`

---

## Pinia Store Design

### `useCharacterBuilderStore`

```typescript
// stores/characterBuilder.ts
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
  const selectedClass = ref<Class | null>(null)
  const selectedBackground = ref<Background | null>(null)
  const selectedSpells = ref<CharacterSpell[]>([])

  // ══════════════════════════════════════════════════════════════
  // COMPUTED STATS (from API)
  // ══════════════════════════════════════════════════════════════
  const characterStats = ref<CharacterStats | null>(null)
  const validationStatus = computed(() =>
    characterStats.value?.validation_status ?? { is_complete: false, missing: [] }
  )

  // ══════════════════════════════════════════════════════════════
  // DERIVED STATE
  // ══════════════════════════════════════════════════════════════
  const isCaster = computed(() =>
    selectedClass.value?.spellcasting_ability !== null
  )
  const isComplete = computed(() =>
    validationStatus.value.is_complete
  )
  const racialBonuses = computed(() =>
    selectedRace.value?.modifiers?.filter(m => m.modifier_category === 'ability_score') ?? []
  )

  // ══════════════════════════════════════════════════════════════
  // ACTIONS
  // ══════════════════════════════════════════════════════════════

  // Step 1: Create draft character
  async function createDraft(characterName: string): Promise<void> {
    const response = await apiFetch<{ data: Character }>('/characters', {
      method: 'POST',
      body: { name: characterName }
    })
    characterId.value = response.data.id
    name.value = characterName
  }

  // Step 2: Select race
  async function selectRace(race: Race, subrace?: Race): Promise<void> {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: {
        race_id: subrace?.id ?? race.id
      }
    })
    raceId.value = race.id
    subraceId.value = subrace?.id ?? null
    selectedRace.value = subrace ?? race
    await refreshStats()
  }

  // Step 3: Select class
  async function selectClass(cls: Class): Promise<void> {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: { class_id: cls.id }
    })
    classId.value = cls.id
    selectedClass.value = cls
    await refreshStats()
  }

  // Step 4: Set ability scores
  async function setAbilityScores(scores: AbilityScores): Promise<void> {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: {
        strength: scores.strength,
        dexterity: scores.dexterity,
        constitution: scores.constitution,
        intelligence: scores.intelligence,
        wisdom: scores.wisdom,
        charisma: scores.charisma
      }
    })
    abilityScores.value = { ...scores }
    await refreshStats()
  }

  // Step 5: Select background
  async function selectBackground(background: Background): Promise<void> {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: { background_id: background.id }
    })
    backgroundId.value = background.id
    selectedBackground.value = background
    await refreshStats()
  }

  // Step 6: Learn a spell
  async function learnSpell(spellId: number): Promise<void> {
    const response = await apiFetch<{ data: CharacterSpell }>(
      `/characters/${characterId.value}/spells`,
      {
        method: 'POST',
        body: { spell_id: spellId }
      }
    )
    selectedSpells.value.push(response.data)
  }

  // Step 6: Forget a spell
  async function forgetSpell(spellId: number): Promise<void> {
    await apiFetch(`/characters/${characterId.value}/spells/${spellId}`, {
      method: 'DELETE'
    })
    selectedSpells.value = selectedSpells.value.filter(s => s.spell.id !== spellId)
  }

  // Refresh stats from API
  async function refreshStats(): Promise<void> {
    if (!characterId.value) return
    const response = await apiFetch<{ data: CharacterStats }>(
      `/characters/${characterId.value}/stats`
    )
    characterStats.value = response.data
  }

  // Navigation
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

  // Reset wizard
  function reset(): void {
    characterId.value = null
    name.value = ''
    raceId.value = null
    subraceId.value = null
    classId.value = null
    backgroundId.value = null
    abilityScores.value = {
      strength: 10, dexterity: 10, constitution: 10,
      intelligence: 10, wisdom: 10, charisma: 10
    }
    selectedRace.value = null
    selectedClass.value = null
    selectedBackground.value = null
    selectedSpells.value = []
    characterStats.value = null
    currentStep.value = 1
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
    characterStats,
    validationStatus,
    isCaster,
    isComplete,
    racialBonuses,
    // Actions
    createDraft,
    selectRace,
    selectClass,
    setAbilityScores,
    selectBackground,
    learnSpell,
    forgetSpell,
    refreshStats,
    nextStep,
    previousStep,
    goToStep,
    reset
  }
})
```

### Store Design Decisions

1. **Session-only state** - No IndexedDB persistence. Character is saved to API immediately.
2. **Dual storage** - Store both IDs (for API) and full entities (for display)
3. **API-first computed stats** - Backend calculates all D&D math via `CharacterStatCalculator`
4. **Optimistic updates** - Update local state, then sync with API

---

## Component Structure

```
app/
├── pages/
│   └── characters/
│       ├── index.vue                 # Character list
│       ├── create.vue                # Wizard container page
│       └── [id]/
│           └── index.vue             # Character sheet (view)
│
├── components/
│   └── character/
│       ├── builder/
│       │   ├── Stepper.vue           # UStepper wrapper
│       │   ├── StepName.vue          # Step 1: Name input
│       │   ├── StepRace.vue          # Step 2: Race selection
│       │   ├── StepClass.vue         # Step 3: Class selection
│       │   ├── StepAbilities.vue     # Step 4: Ability scores
│       │   ├── StepBackground.vue    # Step 5: Background
│       │   ├── StepSpells.vue        # Step 6: Spells (conditional)
│       │   ├── StepReview.vue        # Step 7: Final review
│       │   ├── StatPreview.vue       # Live stats sidebar
│       │   ├── RaceCard.vue          # Race selection card
│       │   ├── ClassCard.vue         # Class selection card
│       │   ├── BackgroundCard.vue    # Background selection card
│       │   ├── AbilityScoreInput.vue # Single ability input
│       │   └── SpellCheckbox.vue     # Spell selection checkbox
│       │
│       ├── sheet/
│       │   ├── Header.vue            # Name, race, class, level
│       │   ├── AbilityScores.vue     # 6-stat block
│       │   ├── CombatStats.vue       # AC, HP, Initiative
│       │   ├── SavingThrows.vue      # Save bonuses
│       │   ├── Skills.vue            # Skill list with proficiencies
│       │   ├── Features.vue          # Racial traits, class features
│       │   └── Spells.vue            # Spell list
│       │
│       ├── Card.vue                  # Character card for list
│       └── EmptyState.vue            # No characters message
│
├── composables/
│   └── useCharacterBuilder.ts        # Wizard logic helpers
│
└── stores/
    └── characterBuilder.ts           # Wizard state
```

---

## Step Implementations

### Step 1: Name

Simple text input with validation.

```vue
<!-- character/builder/StepName.vue -->
<template>
  <div class="space-y-6">
    <div class="text-center">
      <h2 class="text-2xl font-bold">Name Your Character</h2>
      <p class="text-gray-500 mt-2">
        What shall this hero be called?
      </p>
    </div>

    <UFormField label="Character Name" required>
      <UInput
        v-model="characterName"
        placeholder="Enter a name..."
        size="xl"
        class="max-w-md mx-auto"
        autofocus
      />
    </UFormField>

    <div class="flex justify-center">
      <UButton
        :disabled="!isValid"
        :loading="isCreating"
        size="lg"
        @click="handleCreate"
      >
        Begin Your Journey
      </UButton>
    </div>
  </div>
</template>
```

**Validation:** Non-empty, max 255 characters
**API Call:** `POST /characters` with `{ name }`

---

### Step 2: Race Selection

Grid of race cards with detail panel and subrace selection.

**Layout:**
- Search input + optional size filter
- Grid of `RaceCard` components (4 columns desktop, 2 mobile)
- Selected race expands to show full details
- Subrace radio buttons appear if race has subraces

**Data Source:** `GET /races?per_page=100` (cached)

**Key Features:**
- Show ability modifiers on each card (+2 DEX, etc.)
- Highlight key racial traits (Darkvision, Lucky, etc.)
- Subrace selection required before proceeding

---

### Step 3: Class Selection

Similar grid layout to race selection.

**Layout:**
- Search input
- Grid of `ClassCard` components
- Selected class shows hit die, proficiencies, spellcasting info

**Data Source:** `GET /classes?filter=is_base_class=true&per_page=50`

**Key Features:**
- Show hit die prominently (d6, d8, d10, d12)
- Indicate spellcasting ability if present
- Show saving throw proficiencies
- Display archetype name (e.g., "Sorcerous Origin at level 1")

---

### Step 4: Ability Scores

Six number inputs with live modifier calculation.

**Layout:**
```
┌─────────┬─────────┬─────────┬─────────┬─────────┬─────────┐
│   STR   │   DEX   │   CON   │   INT   │   WIS   │   CHA   │
│  [16]   │  [10]   │  [14]   │  [ 8]   │  [12]   │  [15]   │
│  (+3)   │  (+0)   │  (+2)   │  (-1)   │  (+1)   │  (+2)   │
│         │         │         │         │         │   ⭐    │
└─────────┴─────────┴─────────┴─────────┴─────────┴─────────┘
```

**Key Features:**
- Number input with +/- stepper buttons
- Live modifier display: `floor((score - 10) / 2)`
- Star icon on spellcasting ability
- Racial bonuses shown separately below
- Class-specific tips (e.g., "Paladins benefit from STR and CHA")

**Validation:** Each score must be 3-20

---

### Step 5: Background Selection

Grid of background cards.

**Layout:**
- Search input
- Grid of `BackgroundCard` components
- Selected background shows feature and skill proficiencies

**Data Source:** `GET /backgrounds?per_page=100`

**Key Features:**
- Show feature name prominently
- List skill proficiencies granted
- Display languages granted (if any)

---

### Step 6: Spell Selection (Conditional)

Checkbox grid grouped by spell level.

**Visibility:** Only shown if `selectedClass.spellcasting_ability !== null`

**Layout:**
```
CANTRIPS (Choose 3)                     Selected: 2/3
─────────────────────────────────────────────────────
☑ Fire Bolt    ☑ Mage Hand    ☐ Prestidigitation

1ST-LEVEL SPELLS (Choose 6)             Selected: 4/6
─────────────────────────────────────────────────────
☑ Magic Missile  ☑ Shield  ☑ Detect Magic  ☑ Mage Armor
```

**Data Source:** `GET /characters/{id}/available-spells?max_level=1`

**Key Features:**
- Group by spell level (cantrips, 1st, 2nd, etc.)
- Show selection limits from class table
- Click spell to see details in side panel
- Disable checkbox when limit reached

**Spell Limit Handling:**
- API returns limits based on class/level
- Frontend enforces selection count
- Always-prepared spells (domain, etc.) shown but not selectable

---

### Step 7: Review

Final character sheet preview.

**Layout:**
- Full character sheet using `sheet/*` components
- Validation status indicator (complete/incomplete)
- "Create Character" button

**Data Source:** `GET /characters/{id}/stats`

**Key Features:**
- All computed stats displayed
- Missing fields highlighted
- Option to go back and edit any step
- Final creation confirmation

---

## Stat Preview Sidebar

Persistent sidebar showing live character stats.

**Visibility:** Steps 2-7 (after character is created)

**Content:**
```
┌─────────────────────────┐
│ PREVIEW                 │
├─────────────────────────┤
│ Thorin Ironforge        │
│ Dwarf Fighter           │
│ Level 1                 │
├─────────────────────────┤
│ STR 16 (+3)  DEX 10 (+0)│
│ CON 14 (+2)  INT  8 (-1)│
│ WIS 12 (+1)  CHA 10 (+0)│
├─────────────────────────┤
│ AC: 16   HP: 12         │
│ Prof Bonus: +2          │
├─────────────────────────┤
│ Missing:                │
│ • Background            │
└─────────────────────────┘
```

**Updates:** Refreshes after each API call via `refreshStats()`

---

## Mobile Considerations

1. **Stepper collapses** - Show current step number, not full stepper
2. **Cards stack vertically** - 1 column on mobile
3. **Preview as bottom sheet** - Slide up from bottom instead of sidebar
4. **Larger touch targets** - All inputs/buttons minimum 44px
5. **Sticky navigation** - Back/Next buttons fixed at bottom

---

## Error Handling

### API Errors

```typescript
// In store actions
async function selectRace(race: Race): Promise<void> {
  try {
    await apiFetch(...)
  } catch (error) {
    // Show toast notification
    useToast().add({
      title: 'Failed to save race selection',
      description: error.message,
      color: 'red'
    })
    throw error // Let component handle UI state
  }
}
```

### Validation Errors

- Field-level validation before API call
- API validation errors mapped to form fields
- Global validation status from `validation_status.missing`

### Network Errors

- Retry logic in `apiFetch` composable
- Optimistic updates with rollback on failure
- Offline detection with "save when online" queue (future)

---

## Testing Strategy

### Component Tests

| Component | Test Focus |
|-----------|------------|
| `StepName` | Input validation, create button disabled state |
| `StepRace` | Race card rendering, subrace visibility, selection |
| `StepAbilities` | Score validation (3-20), modifier calculation |
| `StepSpells` | Checkbox limits, grouped display |
| `Stepper` | Step navigation, disabled state |
| `StatPreview` | Computed stat display, missing fields |

### Store Tests

| Action | Test Focus |
|--------|------------|
| `createDraft` | API call, state update |
| `selectRace` | API call, selectedRace update, stats refresh |
| `setAbilityScores` | Validation, API call, local state |
| `learnSpell` | Add to array, API call |
| `reset` | All state cleared |

### E2E Tests

1. Complete wizard flow (all 7 steps)
2. Non-caster flow (skip spell step)
3. Back navigation preserves state
4. Mobile viewport flow

---

## Implementation Phases

### Phase 1: Foundation (3-4 hours)

- [ ] Create Pinia store `characterBuilder.ts`
- [ ] Create pages: `characters/index.vue`, `characters/create.vue`
- [ ] Create `character/builder/Stepper.vue` with step definitions
- [ ] Create `StepName.vue` with character creation
- [ ] Create `StatPreview.vue` sidebar component

### Phase 2: Selection Steps (4-5 hours)

- [ ] Create `StepRace.vue` with race grid and subrace handling
- [ ] Create `RaceCard.vue` component
- [ ] Create `StepClass.vue` with class grid
- [ ] Create `ClassCard.vue` component
- [ ] Create `StepBackground.vue` with background grid
- [ ] Create `BackgroundCard.vue` component

### Phase 3: Ability Scores (2-3 hours)

- [ ] Create `StepAbilities.vue` with 6 inputs
- [ ] Create `AbilityScoreInput.vue` component
- [ ] Add modifier calculation display
- [ ] Add racial bonus preview
- [ ] Add class-specific tips

### Phase 4: Spells (3-4 hours)

- [ ] Create `StepSpells.vue` with conditional visibility
- [ ] Create `SpellCheckbox.vue` component
- [ ] Implement cantrip/spell grouping
- [ ] Add selection limit enforcement
- [ ] Add spell detail panel

### Phase 5: Review & Polish (2-3 hours)

- [ ] Create `StepReview.vue` with full preview
- [ ] Create `character/sheet/*` components
- [ ] Add mobile responsive styles
- [ ] Add loading states and error handling
- [ ] Write tests

### Phase 6: Character List & Sheet (2-3 hours)

- [ ] Create `characters/index.vue` list page
- [ ] Create `character/Card.vue` for list display
- [ ] Create `characters/[id]/index.vue` sheet page
- [ ] Reuse `sheet/*` components

**Total Estimated: 16-22 hours**

---

## Future Enhancements (v2+)

1. **Point Buy / Standard Array** - Issue #87
2. **Multiclass Support** - Backend v2
3. **Character Editing** - Reuse wizard components
4. **Level Up Flow** - Backend v1.5
5. **Equipment Selection** - Starting equipment from class/background
6. **PDF Export** - Generate character sheet PDF
7. **Character Sharing** - Shareable links for characters

---

## Related Documents

- Backend API Design: `importer/docs/plans/2025-11-30-character-builder-api-design.md`
- Backend Implementation: `importer/docs/plans/2025-11-30-character-builder-implementation-plan.md`
- Point Buy Issue: #87

---

**Status:** Ready for Implementation
