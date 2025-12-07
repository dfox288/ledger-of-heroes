# ModifiersDisplay: Choice Modifier Support

**Date:** 2025-11-21
**Status:** Approved
**Component:** `app/components/ui/ModifiersDisplay.vue`

## Overview

Enhance the ModifiersDisplay component to support the new choice-based modifier properties from the API (`is_choice`, `choice_count`, `choice_constraint`).

## Problem

The API now returns choice-based modifiers (e.g., Half-Elf's "choose 2 different ability scores for +1") but the component doesn't display this information. Users can't distinguish between automatic modifiers and those requiring character creation choices.

## Design Decision

**Approach:** Inline choice description with badge + descriptive text

**Why this approach:**
- Simple implementation, minimal refactoring
- Clear communication of player choices
- Maintains existing component structure
- Accessible and mobile-friendly
- Easy to test

**Rejected alternatives:**
- Separate sections (too complex, breaks tests)
- Tooltip/popover (hides critical information)

## API Changes

### New Modifier Properties
```typescript
interface Modifier {
  // Existing
  id: number
  modifier_category: string
  ability_score?: AbilityScore | null
  value: string | number
  condition?: string | null

  // NEW
  is_choice: boolean
  choice_count: number | null
  choice_constraint: string | null
}
```

### Example Data

**Fixed Modifier (Human):**
```json
{
  "modifier_category": "ability_score",
  "ability_score": {"code": "STR", "name": "Strength"},
  "value": "1",
  "is_choice": false,
  "choice_count": null,
  "choice_constraint": null
}
```

**Choice Modifier (Half-Elf):**
```json
{
  "modifier_category": "ability_score",
  "value": "+1",
  "is_choice": true,
  "choice_count": 2,
  "choice_constraint": "different"
}
```

## UI Design

### Visual Layout

**Choice Modifier Display:**
```
┌─────────────────────────────────────┐
│ [CHOICE]                            │
│ Choose 2 different ability scores   │
│ +1 each                             │
└─────────────────────────────────────┐
```

**Fixed Modifier Display (unchanged):**
```
┌─────────────────────────────────────┐
│ Strength (STR): +1                  │
└─────────────────────────────────────┘
```

### Components Used
- `<UBadge>` with `color="info"` and `variant="soft"` for choice indicator
- Existing card styling and spacing

### Choice Description Generation

**Format Logic:**
- `choice_count + choice_constraint` → description
- Handle singular/plural: "ability score" vs "ability scores"
- Handle constraints: "different", null, or other values

**Examples:**
| Count | Constraint | Output |
|-------|-----------|---------|
| 1 | null | "Choose 1 ability score" |
| 2 | "different" | "Choose 2 different ability scores" |
| 3 | "different" | "Choose 3 different ability scores" |
| 1 | "different" | "Choose 1 ability score" |

**Value Display:**
- For choices without specific ability score: show value after description
- For choices with ability score (rare): show ability + value
- Format: "+1 each" or just the value

## Implementation Details

### Helper Function
```typescript
const formatChoiceDescription = (
  count: number | null,
  constraint: string | null,
  category: string
): string => {
  if (!count) return `Choose ${category}`

  const plural = count > 1 ? 's' : ''
  const constraintText = constraint ? `${constraint} ` : ''
  const target = category === 'ability_score'
    ? `ability score${plural}`
    : category

  return `Choose ${count} ${constraintText}${target}`
}
```

### Template Changes
```vue
<div v-if="modifier.is_choice" class="space-y-2">
  <UBadge color="info" variant="soft" size="sm">CHOICE</UBadge>
  <div class="font-medium">
    {{ formatChoiceDescription(...) }}
  </div>
  <div class="text-sm text-gray-600 dark:text-gray-400">
    {{ formatValue(modifier.value) }} each
  </div>
</div>
```

## Testing Strategy

### New Test Cases (6 tests minimum)
1. ✅ Choice modifier with count=2 and constraint="different" displays correctly
2. ✅ Choice modifier with count=1 and no constraint displays correctly
3. ✅ Choice badge only appears when is_choice=true
4. ✅ Choice description handles singular/plural correctly
5. ✅ Mixed modifiers (choice + fixed) both display properly
6. ✅ Existing 10 tests continue passing

### Test Data Examples
```typescript
// Choice modifier
{
  id: 1,
  modifier_category: 'ability_score',
  value: '+1',
  is_choice: true,
  choice_count: 2,
  choice_constraint: 'different'
}

// Fixed modifier
{
  id: 2,
  modifier_category: 'ability_score',
  ability_score: { id: 1, code: 'STR', name: 'Strength' },
  value: 2,
  is_choice: false,
  choice_count: null,
  choice_constraint: null
}
```

## Success Criteria

- ✅ All 10 existing tests pass (no regressions)
- ✅ 6+ new tests added and passing
- ✅ Choice modifiers display with badge + description
- ✅ Fixed modifiers display unchanged
- ✅ Component handles missing/null values gracefully
- ✅ Works in both light and dark mode
- ✅ Verified with actual API data (Half-Elf race)

## Files Changed

- `app/components/ui/ModifiersDisplay.vue` - component implementation
- `tests/components/ui/ModifiersDisplay.test.ts` - test updates

## Related Issues

This addresses the technical debt where new API properties weren't reflected in the UI, causing confusion about which modifiers are automatic vs player choices.
