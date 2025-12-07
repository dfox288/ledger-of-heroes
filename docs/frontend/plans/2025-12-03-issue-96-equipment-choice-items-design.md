# Design: Equipment Choice Items (Issue #96)

**Date:** 2025-12-03
**Status:** Approved
**Issue:** [#96 - Add structured item type references for equipment choices](https://github.com/dfox288/dnd-rulebook-project/issues/96)

## Problem

Equipment choices like "a martial weapon and a shield" currently only display text descriptions. Users cannot select their actual starting weapon (e.g., longsword vs battleaxe) because the frontend lacks structured data linking choices to item categories.

## Solution

The backend now provides a `choice_items` array on `EntityItemResource` that breaks down compound equipment choices into:
- **Category items** (`proficiency_type` set, `item` null) - user must pick from matching items
- **Fixed items** (`proficiency_type` null, `item` set) - auto-included, no selection needed

The frontend will render inline item pickers for category selections.

## API Changes (Backend Complete)

### New Type: `EquipmentChoiceItemResource`

```typescript
interface EquipmentChoiceItemResource {
  proficiency_type?: ProficiencyTypeResource  // Category filter (e.g., "Martial Weapons")
  item?: ItemResource                          // Specific item (e.g., Shield)
  quantity: number                             // How many needed
}
```

### Updated Type: `EntityItemResource`

```typescript
interface EntityItemResource {
  // ... existing fields ...
  choice_items?: EquipmentChoiceItemResource[]  // NEW: sub-items within this choice
}
```

### Example API Response

```json
{
  "description": "a martial weapon and a shield",
  "choice_items": [
    { "proficiency_type": { "slug": "martial-weapons", "subcategory": "martial" }, "item": null, "quantity": 1 },
    { "proficiency_type": null, "item": { "id": 48, "name": "Shield" }, "quantity": 1 }
  ]
}
```

### Equipment Endpoints

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/characters/{id}/equipment` | List character's equipment |
| POST | `/characters/{id}/equipment` | Add item (`{ item_id, quantity }`) |
| DELETE | `/characters/{id}/equipment/{id}` | Remove item |

## Frontend Design

### Approach: Inline Expansion

When user selects a choice option with category items, pickers appear inline below the selection:

```
Equipment Choice 2
┌─────────────────────────────────────────────────────────┐
│ ● a martial weapon and a shield                        │
│   └─ Select martial weapon: [Longsword ▼]              │
│   └─ Shield ✓ (auto-included)                          │
│                                                        │
│ ○ two martial weapons                                  │
└─────────────────────────────────────────────────────────┘
```

### Component Architecture

```
StepEquipment.vue (existing)
├── EquipmentChoiceGroup.vue (enhanced)
│   ├── Radio buttons for each choice_option
│   └── Inline pickers when option has category choice_items
│       └── EquipmentItemPicker.vue (new)
│           ├── Fetches items via /items?filter=...
│           ├── USelectMenu dropdown
│           └── Emits selected item_id
```

### New Component: `EquipmentItemPicker.vue`

**Props:**
- `proficiencyType: ProficiencyTypeResource` - category to filter by
- `quantity: number` - how many to select (1 or 2)
- `modelValue: number[]` - selected item IDs (v-model)
- `disabled: boolean` - whether picker is active

**Emits:**
- `update:modelValue` - when selection changes

**Item Filtering:**
```typescript
// Build filter from proficiency type
function buildItemFilter(proficiencyType: ProficiencyTypeResource): string {
  const subcategory = proficiencyType.subcategory

  // Weapons need both melee and ranged variants
  if (subcategory === 'martial' || subcategory === 'simple') {
    return `proficiency_category IN [${subcategory}_melee, ${subcategory}_ranged] AND is_magic = false`
  }

  return `proficiency_category = ${subcategory} AND is_magic = false`
}
```

**Constraint:** `is_magic = false` ensures only mundane items for starting equipment.

### State Management

**New store state:**
```typescript
// Track item selections within compound choices
// Key: "choice_group:choice_option:choice_item_index"
const equipmentItemSelections = ref<Map<string, number>>(new Map())
```

**Helper functions:**
```typescript
function makeSelectionKey(choiceGroup: string, choiceOption: number, index: number): string {
  return `${choiceGroup}:${choiceOption}:${index}`
}

function setEquipmentItemSelection(key: string, itemId: number): void {
  equipmentItemSelections.value.set(key, itemId)
}

function getEquipmentItemSelection(key: string): number | undefined {
  return equipmentItemSelections.value.get(key)
}
```

**Enhanced validation:**
```typescript
const allEquipmentChoicesMade = computed(() => {
  for (const [group] of equipmentByChoiceGroup.value) {
    if (!equipmentChoices.value.has(group)) return false

    const selectedOptionId = equipmentChoices.value.get(group)
    const selectedOption = findOptionById(selectedOptionId)

    // Check all category items have selections
    for (const [index, choiceItem] of (selectedOption?.choice_items ?? []).entries()) {
      if (choiceItem.proficiency_type && !choiceItem.item) {
        const key = makeSelectionKey(group, selectedOption.choice_option!, index)
        for (let i = 0; i < choiceItem.quantity; i++) {
          if (!equipmentItemSelections.value.has(`${key}:${i}`)) return false
        }
      }
    }
  }
  return true
})
```

### API Integration

**New Nitro routes:**
- `server/api/characters/[id]/equipment.get.ts`
- `server/api/characters/[id]/equipment.post.ts`
- `server/api/characters/[id]/equipment/[equipmentId].delete.ts`

**Store action:**
```typescript
async function saveEquipmentChoices(): Promise<void> {
  for (const [group, optionId] of equipmentChoices.value) {
    const option = findOptionById(optionId)

    for (const [index, choiceItem] of (option?.choice_items ?? []).entries()) {
      let itemId: number
      let quantity = choiceItem.quantity

      if (choiceItem.item) {
        itemId = choiceItem.item.id
      } else {
        const key = makeSelectionKey(group, option!.choice_option!, index)
        itemId = equipmentItemSelections.value.get(`${key}:0`)!
        // Handle quantity > 1 (multiple selections)
      }

      await apiFetch(`/characters/${characterId.value}/equipment`, {
        method: 'POST',
        body: { item_id: itemId, quantity }
      })
    }
  }
}
```

## UI Flow

1. User reaches StepEquipment
2. Sees choice groups with radio options (existing)
3. Selects an option → if it has `choice_items`:
   - Fixed items show with ✓ checkmark (auto-included)
   - Category items show dropdown picker
4. Continue button disabled until all selections made
5. On continue → persists to `/characters/{id}/equipment` → next step

## Testing Strategy

- Unit tests for `EquipmentItemPicker` component
- Unit tests for enhanced `EquipmentChoiceGroup`
- Integration tests for full StepEquipment flow
- Store tests for new state management

## Files to Create/Modify

### New Files
- `app/components/character/builder/EquipmentItemPicker.vue`
- `server/api/characters/[id]/equipment.get.ts`
- `server/api/characters/[id]/equipment.post.ts`
- `server/api/characters/[id]/equipment/[equipmentId].delete.ts`
- `tests/components/character/builder/EquipmentItemPicker.test.ts`

### Modified Files
- `app/components/character/builder/EquipmentChoiceGroup.vue`
- `app/components/character/builder/StepEquipment.vue`
- `app/stores/characterBuilder.ts`
- `tests/fixtures/equipment.ts`
- `tests/components/character/builder/EquipmentChoiceGroup.test.ts`
- `tests/components/character/builder/StepEquipment.test.ts`
