# Issue #106: Character Proficiencies & Features Integration

**Date:** 2025-12-03
**Issue:** [#106](https://github.com/dfox288/dnd-rulebook-project/issues/106)
**Status:** Design Complete

## Overview

Integrate backend proficiency and feature endpoints into the character builder wizard. The backend auto-populates proficiencies when `class_id`, `race_id`, or `background_id` are set, but some selections require user input (e.g., "choose 2 skills from this list").

## Backend API Summary

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/characters/{id}/proficiencies` | GET | List all granted proficiencies |
| `/characters/{id}/proficiency-choices` | GET | Get pending choices (skills to pick) |
| `/characters/{id}/proficiency-choices` | POST | Submit skill selections |
| `/characters/{id}/features` | GET | List all features (class, race, background) |

### Proficiency Choices Response Structure

```json
{
  "data": {
    "class": {
      "skill_choice_1": {
        "quantity": 3,
        "remaining": 3,
        "options": [
          { "type": "skill", "skill_id": 1, "skill": { "id": 1, "name": "Acrobatics", "slug": "acrobatics" } }
        ]
      }
    },
    "race": {},
    "background": {}
  }
}
```

### Submit Choices Request

```json
{
  "source": "class",
  "choice_group": "skill_choice_1",
  "skill_ids": [1, 5, 10]
}
```

## Design Decisions

### 1. Consolidated Step (not scattered)

Proficiency choices appear in a **single new step** after Background selection, rather than mini-steps after each source. This simplifies the implementation and groups all choices together.

### 2. Conditional Step (like Spells)

The Proficiency Choices step only appears if `hasPendingChoices` is true. Characters with no choices (e.g., Human Fighter with Soldier background) skip directly to Equipment.

### 3. Step Order

```
1. Name
2. Race
3. Class
4. Abilities
5. Background
6. Proficiency Choices (NEW - conditional)
7. Equipment
8. Spells (conditional - casters only)
9. Review (enhanced with proficiencies & features display)
```

## Implementation Plan

### Part 1: Nitro Server Routes

Create 4 new proxy routes:

```
server/api/characters/[id]/
├── proficiencies.get.ts
├── proficiency-choices.get.ts
├── proficiency-choices.post.ts
└── features.get.ts
```

### Part 2: Store Changes (`characterBuilder.ts`)

**New State:**
```typescript
// API response for pending choices
const proficiencyChoices = ref<ProficiencyChoicesResponse | null>(null)

// User selections before save: Map<"source:choice_group", Set<skillId>>
const pendingProficiencySelections = ref<Map<string, Set<number>>>(new Map())
```

**New Computed:**
```typescript
// Check if any choices are pending
const hasPendingChoices = computed(() => {
  if (!proficiencyChoices.value) return false
  const { class: cls, race, background } = proficiencyChoices.value.data
  return Object.keys(cls).length > 0 ||
         Object.keys(race).length > 0 ||
         Object.keys(background).length > 0
})

// Update totalSteps to include proficiency step
const totalSteps = computed(() => {
  let steps = 7 // Base steps
  if (hasPendingChoices.value) steps++
  if (isCaster.value) steps++
  return steps
})
```

**New Actions:**
```typescript
// Fetch choices after background selection
async function fetchProficiencyChoices(): Promise<void> {
  const response = await apiFetch<ProficiencyChoicesResponse>(
    `/characters/${characterId.value}/proficiency-choices`
  )
  proficiencyChoices.value = response
}

// Toggle skill selection in local state
function toggleProficiencySelection(
  source: 'class' | 'race' | 'background',
  choiceGroup: string,
  skillId: number
): void {
  const key = `${source}:${choiceGroup}`
  const current = pendingProficiencySelections.value.get(key) ?? new Set()
  if (current.has(skillId)) {
    current.delete(skillId)
  } else {
    current.add(skillId)
  }
  pendingProficiencySelections.value.set(key, current)
}

// Save all selections to API
async function saveProficiencyChoices(): Promise<void> {
  for (const [key, skillIds] of pendingProficiencySelections.value) {
    const [source, choiceGroup] = key.split(':')
    await apiFetch(`/characters/${characterId.value}/proficiency-choices`, {
      method: 'POST',
      body: { source, choice_group: choiceGroup, skill_ids: [...skillIds] }
    })
  }
}
```

### Part 3: Step Component (`StepProficiencies.vue`)

**Props:** None (uses store)

**UI Structure:**
- Header: "Choose Your Proficiencies"
- Subheader: "Your class, race, and background grant the following choices"
- For each source with choices:
  - Source header (e.g., "From Class: Bard")
  - Choice group header (e.g., "Choose 3 skills:")
  - Grid of skill checkboxes with ability score indicators
  - Selection counter (e.g., "2/3 selected")
- Footer: Back/Next buttons (Next disabled until all choices complete)

**Validation:**
```typescript
const allChoicesComplete = computed(() => {
  if (!proficiencyChoices.value) return true

  for (const [source, groups] of Object.entries(proficiencyChoices.value.data)) {
    for (const [groupName, group] of Object.entries(groups)) {
      const key = `${source}:${groupName}`
      const selected = pendingProficiencySelections.value.get(key)?.size ?? 0
      if (selected !== group.quantity) return false
    }
  }
  return true
})
```

### Part 4: Edit Page Updates (`[id]/edit.vue`)

Update step rendering logic to handle the new conditional step:

```typescript
const currentStepComponent = computed(() => {
  const step = store.currentStep

  // Steps 1-5 are fixed
  if (step <= 5) return stepComponents[step - 1]

  // Step 6+ depends on conditions
  let offset = 5

  if (store.hasPendingChoices) {
    offset++
    if (step === offset) return StepProficiencies
  }

  offset++ // Equipment is always present
  if (step === offset) return StepEquipment

  if (store.isCaster) {
    offset++
    if (step === offset) return StepSpells
  }

  return StepReview // Last step
})
```

### Part 5: Review Step Enhancement

Add two new sections to `StepReview.vue`:

**Proficiencies Section:**
- Fetch from `GET /characters/{id}/proficiencies`
- Display grouped by type: Skills, Saving Throws, Armor, Weapons, Tools, Languages
- Edit button navigates to proficiency step

**Features Section:**
- Fetch from `GET /characters/{id}/features`
- Display grouped by source: Class Features, Racial Traits, Background Feature
- Collapsible descriptions for each feature

## TypeScript Types

```typescript
interface ProficiencyChoice {
  quantity: number
  remaining: number
  options: Array<{
    type: 'skill'
    skill_id: number
    skill: {
      id: number
      name: string
      slug: string
    }
  }>
}

interface ProficiencyChoicesResponse {
  data: {
    class: Record<string, ProficiencyChoice>
    race: Record<string, ProficiencyChoice>
    background: Record<string, ProficiencyChoice>
  }
}

interface CharacterProficiency {
  id: number
  source: 'class' | 'race' | 'background'
  type: 'skill' | 'saving_throw' | 'armor' | 'weapon' | 'tool' | 'language'
  name: string
  // Additional fields based on type
}

interface CharacterFeature {
  id: number
  source: 'class' | 'race' | 'background'
  name: string
  description: string
  level?: number // For class features
}
```

## Testing Strategy

### Unit Tests
- `StepProficiencies.test.ts`: Selection toggle, validation, save flow
- `characterBuilder.test.ts`: New actions and computed properties
- Nitro route tests (if applicable)

### Integration Tests
- Full wizard flow with class that has skill choices
- Wizard flow with class that has NO skill choices (step skipped)
- Edit mode: Loading existing proficiency selections

## File Changes Summary

| File | Change |
|------|--------|
| `server/api/characters/[id]/proficiencies.get.ts` | NEW |
| `server/api/characters/[id]/proficiency-choices.get.ts` | NEW |
| `server/api/characters/[id]/proficiency-choices.post.ts` | NEW |
| `server/api/characters/[id]/features.get.ts` | NEW |
| `app/stores/characterBuilder.ts` | ADD state, computed, actions |
| `app/components/character/builder/StepProficiencies.vue` | NEW |
| `app/components/character/builder/StepReview.vue` | ADD proficiencies & features sections |
| `app/pages/characters/[id]/edit.vue` | UPDATE step routing logic |
| `tests/components/character/builder/StepProficiencies.test.ts` | NEW |
| `tests/stores/characterBuilder.test.ts` | ADD tests for new functionality |

## Success Criteria

- [ ] Proficiency choices step appears only when needed
- [ ] All skill choices can be selected with proper validation
- [ ] Selections persist to backend via POST
- [ ] Review step shows all proficiencies grouped by type
- [ ] Review step shows all features grouped by source
- [ ] Edit mode loads existing selections correctly
- [ ] Step numbering is correct with all conditional combinations
