# Design: Ability Score Modifier Choices (Issue #219)

**Date:** 2025-12-08
**Status:** Approved
**Related Issues:** #219 (frontend), #352 (backend - completed)

## Overview

Races like Half-Elf have choice-based ability score modifiers (e.g., "Choose 2 different abilities for +1 each"). This design adds UI support for these choices in `StepAbilities.vue`.

## Problem

Currently `StepAbilities.vue` shows a warning placeholder for races with ability score choices. The backend now supports this via the unified pending-choices API (#352 merged), but the frontend lacks the selection UI.

## Solution: Inline Checkbox Grid

Integrate ability score choice selection directly into the Racial Bonuses section of `StepAbilities.vue`, using the same pattern as `StepLanguages.vue`.

### Why This Approach

1. **Consistent with existing patterns** - `StepLanguages` uses checkbox-grid for choices
2. **Immediate feedback** - Users see selections reflected in `finalScores` instantly
3. **Clear constraint handling** - Can disable/dim options with visual explanation
4. **Minimal new components** - Logic added to existing `StepAbilities.vue`

## Technical Design

### 1. Composable Updates

**File:** `app/composables/useUnifiedChoices.ts`

```typescript
// Add ability_score to ChoiceType union
type ChoiceType = 'proficiency' | 'language' | 'equipment' | 'spell' |
                  'subclass' | 'asi_or_feat' | 'optional_feature' |
                  'expertise' | 'fighting_style' | 'hit_points' |
                  'ability_score'  // NEW

// Add to choicesByType computed
const choicesByType = computed(() => ({
  // ... existing types ...
  abilityScores: choices.value.filter(c => c.type === 'ability_score')  // NEW
}))
```

### 2. State Management

**File:** `app/components/character/wizard/StepAbilities.vue`

```typescript
// Local selections: Map<choiceId, Set<abilityCode>>
const abilityScoreSelections = ref<Map<string, Set<string>>>(new Map())

// Use unified choices composable
const { choicesByType, fetchChoices, resolveChoice } = useUnifiedChoices(
  computed(() => store.characterId)
)

// Fetch ability_score choices on mount
onMounted(async () => {
  if (store.characterId) {
    await fetchChoices('ability_score')
  }
})

// Initialize from already-resolved selections
watch(() => choicesByType.value.abilityScores, (choices) => {
  for (const choice of choices) {
    if (!abilityScoreSelections.value.has(choice.id) && choice.selected.length > 0) {
      abilityScoreSelections.value.set(choice.id, new Set(choice.selected.map(String)))
    }
  }
}, { immediate: true })
```

### 3. Validation Logic

```typescript
// Check if ability is selected in this choice
function isAbilitySelected(choiceId: string, code: string): boolean {
  return abilityScoreSelections.value.get(choiceId)?.has(code) ?? false
}

// Check if ability has a fixed racial bonus
function hasFixedBonus(code: string): boolean {
  return fixedBonuses.value.some(b => b.ability_score?.code === code)
}

// Get current selection count
function getSelectionCount(choiceId: string): number {
  return abilityScoreSelections.value.get(choiceId)?.size ?? 0
}

// Handle toggle with validation
function handleAbilityToggle(choice: PendingChoice, code: string) {
  const isSelected = isAbilitySelected(choice.id, code)
  const count = getSelectionCount(choice.id)

  // Can't exceed quantity (unless deselecting)
  if (!isSelected && count >= choice.quantity) return

  // Toggle
  const selections = abilityScoreSelections.value.get(choice.id) ?? new Set()
  const updated = new Set(selections)
  isSelected ? updated.delete(code) : updated.add(code)
  abilityScoreSelections.value.set(choice.id, updated)
}

// Check all choices complete
const allAbilityChoicesComplete = computed(() => {
  for (const choice of choicesByType.value.abilityScores) {
    const count = abilityScoreSelections.value.get(choice.id)?.size ?? 0
    if (count < choice.quantity) return false
  }
  return true
})
```

### 4. UI Template

Replace the warning `<UAlert>` with checkbox grid:

```vue
<!-- Ability Score Choices -->
<div v-for="choice in choicesByType.abilityScores" :key="choice.id" class="mt-4">
  <div class="flex items-center justify-between mb-3">
    <span class="text-sm font-medium">
      Choose {{ choice.quantity }} different
      {{ choice.quantity > 1 ? 'abilities' : 'ability' }} for
      +{{ choice.metadata?.bonus_value }}:
    </span>
    <UBadge
      :color="getSelectionCount(choice.id) === choice.quantity ? 'success' : 'neutral'"
      size="md"
    >
      {{ getSelectionCount(choice.id) }}/{{ choice.quantity }} selected
    </UBadge>
  </div>

  <div class="grid grid-cols-3 md:grid-cols-6 gap-3">
    <button
      v-for="option in choice.options"
      :key="option.code"
      type="button"
      :data-testid="`ability-choice-${option.code}`"
      class="ability-option p-3 rounded-lg border text-center transition-all"
      :class="{
        'border-race bg-race/10': isAbilitySelected(choice.id, option.code),
        'border-gray-200 dark:border-gray-700 hover:border-race/50':
          !isAbilitySelected(choice.id, option.code) && !hasFixedBonus(option.code),
        'opacity-50 cursor-not-allowed': hasFixedBonus(option.code)
      }"
      :disabled="hasFixedBonus(option.code)"
      @click="handleAbilityToggle(choice, option.code)"
    >
      <div class="font-bold">{{ option.code }}</div>
      <div class="text-xs text-gray-500">{{ option.name }}</div>
    </button>
  </div>
</div>
```

### 5. Final Scores Integration

Update `finalScores` computed to include chosen bonuses:

```typescript
const finalScores = computed(() => {
  const base = selectedMethod.value === 'standard_array'
    ? nullableScores.value
    : localScores.value

  const codeMap: Record<string, string> = {
    STR: 'strength', DEX: 'dexterity', CON: 'constitution',
    INT: 'intelligence', WIS: 'wisdom', CHA: 'charisma'
  }
  const reverseCodeMap = Object.fromEntries(
    Object.entries(codeMap).map(([k, v]) => [v, k])
  )

  const result: Record<string, { base: number | null, bonus: number, total: number | null }> = {}

  for (const ability of abilities) {
    const baseScore = base[ability]

    // Fixed racial bonuses
    const fixedBonus = fixedBonuses.value
      .filter(m => codeMap[m.ability_score?.code ?? ''] === ability)
      .reduce((sum, m) => sum + Number(m.value), 0)

    // Chosen racial bonuses
    let chosenBonus = 0
    for (const choice of choicesByType.value.abilityScores) {
      const selections = abilityScoreSelections.value.get(choice.id)
      const abilityCode = reverseCodeMap[ability]
      if (selections?.has(abilityCode)) {
        chosenBonus += Number(choice.metadata?.bonus_value ?? 1)
      }
    }

    const totalBonus = fixedBonus + chosenBonus

    result[ability] = {
      base: baseScore,
      bonus: totalBonus,
      total: baseScore !== null ? baseScore + totalBonus : null
    }
  }
  return result
})
```

### 6. Save Flow

Update `saveAndContinue()`:

```typescript
async function saveAndContinue() {
  if (!canSave.value) return

  // Save base ability scores (existing)
  const scores = selectedMethod.value === 'standard_array'
    ? { strength: nullableScores.value.strength ?? 10, ... }
    : localScores.value
  await store.saveAbilityScores(selectedMethod.value, scores)

  // Resolve ability score choices (NEW)
  for (const [choiceId, selectedCodes] of abilityScoreSelections.value.entries()) {
    if (selectedCodes.size > 0) {
      await resolveChoice(choiceId, {
        selected: Array.from(selectedCodes)
      })
    }
  }

  nextStep()
}
```

Update `canSave`:

```typescript
const canSave = computed(() => {
  if (isLoading.value) return false
  if (!isInputValid.value) return false
  if (!allAbilityChoicesComplete.value) return false  // NEW
  return true
})
```

## Constraints

| Constraint | Implementation |
|------------|----------------|
| `choice_constraint: "different"` | Enforced by backend; UI prevents via single Set per choice |
| `quantity` limit | `handleAbilityToggle` blocks selection when count >= quantity |
| Fixed bonus exclusion | `hasFixedBonus()` disables abilities with existing racial bonus |

## Test Plan

1. **Unit tests for validation logic:**
   - `isAbilitySelected` returns correct state
   - `handleAbilityToggle` respects quantity limit
   - `allAbilityChoicesComplete` accurate for various states

2. **Component tests:**
   - Half-Elf shows 6 ability options
   - CHA disabled (fixed +2 bonus)
   - Selection count badge updates
   - Save button disabled until choices complete
   - `finalScores` reflects chosen bonuses

3. **Integration tests:**
   - Full wizard flow with Half-Elf
   - Choices persisted via API
   - Character stats updated correctly

## Files Changed

| File | Change |
|------|--------|
| `app/composables/useUnifiedChoices.ts` | Add `ability_score` type |
| `app/components/character/wizard/StepAbilities.vue` | Add choice UI and logic |
| `tests/components/character/wizard/StepAbilities.test.ts` | Add choice tests |
