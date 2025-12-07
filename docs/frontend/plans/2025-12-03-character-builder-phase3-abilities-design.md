# Character Builder Phase 3: Ability Scores - Design Document

**Date:** 2025-12-03
**Status:** Approved for Implementation
**Parent Issue:** #89
**Phase 2:** Complete (Race & Class Selection)

---

## Overview

### Goal

Implement Step 4 (Ability Scores) of the character creation wizard, allowing players to assign their six ability scores using one of three methods: Standard Array, Point Buy, or Manual Entry.

### User Outcome

- Players select an ability score method via segmented control
- UI adapts to show method-specific input controls
- Racial bonuses are displayed and automatically applied to final scores
- Validation ensures valid scores before proceeding

---

## Supported Methods

### Standard Array
- Assign predefined values: 15, 14, 13, 12, 10, 8
- Each value used exactly once
- Implemented via 6 dropdowns with mutual exclusion

### Point Buy
- 27 points to spend
- Base scores range 8-15
- Cost table:
  | Score | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 |
  |-------|---|---|----|----|----|----|----|----|
  | Cost  | 0 | 1 | 2  | 3  | 4  | 5  | 7  | 9  |
- Implemented via +/- buttons with point tracking

### Manual Entry
- Direct number inputs
- Range: 3-20 (D&D valid ability score range)
- No point validation (DM discretion for rolled stats)
- Implemented via number inputs with spinners

---

## Component Structure

```
app/components/character/builder/
├── StepAbilities.vue        # Parent: method selector + delegates to child
├── StandardArrayInput.vue   # Dropdown-based assignment
├── PointBuyInput.vue        # +/- buttons with point tracking
└── ManualInput.vue          # Direct number inputs
```

---

## Component Specifications

### StepAbilities.vue (Parent)

**Responsibilities:**
- Render method selector (UButtonGroup or UTabs)
- Conditionally render appropriate input component
- Display racial bonuses section
- Display final calculated scores
- Handle "Save & Continue" action

**Props:** None (uses store directly)

**Template Structure:**
```vue
<template>
  <div>
    <!-- Method Selector -->
    <UButtonGroup v-model="selectedMethod">
      <UButton value="standard_array">Standard Array</UButton>
      <UButton value="point_buy">Point Buy</UButton>
      <UButton value="manual">Manual</UButton>
    </UButtonGroup>

    <!-- Method-specific Input -->
    <StandardArrayInput v-if="selectedMethod === 'standard_array'" v-model="baseScores" />
    <PointBuyInput v-else-if="selectedMethod === 'point_buy'" v-model="baseScores" />
    <ManualInput v-else v-model="baseScores" />

    <!-- Racial Bonuses Display -->
    <RacialBonusesDisplay :race="selectedRace" />

    <!-- Final Scores Summary -->
    <FinalScoresSummary :base-scores="baseScores" :racial-bonuses="racialBonuses" />

    <!-- Continue Button -->
    <UButton :disabled="!isValid" @click="saveAndContinue">
      Save & Continue
    </UButton>
  </div>
</template>
```

### StandardArrayInput.vue

**Props:**
- `modelValue`: Record<AbilityCode, number | null>

**Emits:**
- `update:modelValue`

**Behavior:**
- 6 dropdowns (one per ability)
- Available options: 15, 14, 13, 12, 10, 8
- When a value is selected, remove it from other dropdowns
- Valid when all 6 abilities have values assigned

**UI:**
```
STR: [Select ▼]  DEX: [Select ▼]  CON: [Select ▼]
INT: [Select ▼]  WIS: [Select ▼]  CHA: [Select ▼]
```

### PointBuyInput.vue

**Props:**
- `modelValue`: Record<AbilityCode, number>

**Emits:**
- `update:modelValue`

**Internal State:**
- `pointsSpent`: computed from current scores
- `pointsRemaining`: 27 - pointsSpent

**Behavior:**
- All scores start at 8 (0 points)
- +/- buttons adjust score
- Disable + when score=15 or points exhausted
- Disable - when score=8
- Show point cost for next increase

**UI:**
```
Points Remaining: 17/27

STR: [-] 10 [+] (cost: 2)    DEX: [-] 8 [+] (cost: 1)
CON: [-] 14 [+] (cost: 2)    INT: [-] 8 [+] (cost: 1)
WIS: [-] 12 [+] (cost: 1)    CHA: [-] 8 [+] (cost: 1)
```

### ManualInput.vue

**Props:**
- `modelValue`: Record<AbilityCode, number | null>

**Emits:**
- `update:modelValue`

**Behavior:**
- 6 number inputs
- Range: 3-20
- Valid when all 6 have values in range

**UI:**
```
STR: [___]  DEX: [___]  CON: [___]
INT: [___]  WIS: [___]  CHA: [___]
```

---

## Racial Bonuses Display

Always visible below the input method, shows:

```
┌─────────────────────────────────────────────────┐
│ Racial Bonuses (Dwarf)                          │
│ Constitution +2                                 │
├─────────────────────────────────────────────────┤
│ Final Ability Scores                            │
│                                                 │
│ STR: 14        DEX: 12        CON: 13 +2 = 15   │
│ INT: 10        WIS: 15        CHA: 8            │
└─────────────────────────────────────────────────┘
```

**Data Source:**
```typescript
const racialBonuses = computed(() =>
  selectedRace.value?.modifiers?.filter(m =>
    m.modifier_category === 'ability_score'
  ) ?? []
)
```

---

## Store Actions

### `saveAbilityScores(method, scores): Promise<void>`

```typescript
async function saveAbilityScores(
  method: 'standard_array' | 'point_buy' | 'manual',
  scores: AbilityScores
): Promise<void> {
  isLoading.value = true
  error.value = null

  try {
    await apiFetch(`/characters/${characterId.value}`, {
      method: 'PATCH',
      body: {
        ability_score_method: method,
        strength: scores.strength,
        dexterity: scores.dexterity,
        constitution: scores.constitution,
        intelligence: scores.intelligence,
        wisdom: scores.wisdom,
        charisma: scores.charisma
      }
    })

    abilityScores.value = scores
    abilityScoreMethod.value = method
    await refreshStats()
  } catch (err: unknown) {
    error.value = 'Failed to save ability scores'
    throw err
  } finally {
    isLoading.value = false
  }
}
```

---

## Validation Rules

| Method | Valid When |
|--------|------------|
| Standard Array | All 6 abilities assigned, each value used once |
| Point Buy | All scores 8-15, points spent ≤ 27 |
| Manual | All scores in range 3-20 |

---

## API Contract

### Update Character (PATCH /characters/{id})

**Request:**
```json
{
  "ability_score_method": "point_buy",
  "strength": 14,
  "dexterity": 12,
  "constitution": 15,
  "intelligence": 10,
  "wisdom": 13,
  "charisma": 8
}
```

**Response:** Updated CharacterResource with computed modifiers

---

## Testing Strategy

### Component Tests

| Component | Test Focus |
|-----------|------------|
| `StepAbilities` | Method switching, save action, validation state |
| `StandardArrayInput` | Dropdown exclusion, all-assigned validation |
| `PointBuyInput` | Point calculation, button disable states, limits |
| `ManualInput` | Range validation, input handling |

### Store Tests

| Action | Test Focus |
|--------|------------|
| `saveAbilityScores` | API call params, state updates, error handling |

---

## Implementation Order

1. **Store Action** - Add `saveAbilityScores` to characterBuilder store
2. **ManualInput** - Simplest, validates the pattern
3. **StandardArrayInput** - Dropdown exclusion logic
4. **PointBuyInput** - Most complex, point calculation
5. **StepAbilities** - Parent component wiring
6. **Integration** - Wire to wizard, test full flow

---

## Out of Scope

- Rolling dice (4d6 drop lowest) - use Manual for DM-approved rolls
- Racial ability choice (e.g., Half-Elf +1 to two abilities) - future enhancement
- Feat-based ability increases - handled at level-up

---

## Related Documents

- Phase 2 Design: `docs/plans/2025-12-02-character-builder-phase2-design.md`
- Parent Design: `docs/plans/2025-12-01-character-builder-frontend-design.md`
- Issue: #89
