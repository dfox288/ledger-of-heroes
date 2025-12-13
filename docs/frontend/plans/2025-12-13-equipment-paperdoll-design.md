# Equipment Paperdoll Design

**Issue:** #587
**Date:** 2025-12-13
**Status:** Ready for implementation

## Overview

Rework the inventory sidebar to display all 11 equipment slots as a visual paperdoll (character silhouette with slot positions). Add slot picker modal for equipping items with ambiguous slots.

## Layout Changes

### Page Grid

```
Current:  grid-cols-[1fr_280px]  (flexible + 280px sidebar)
New:      grid-cols-[2fr_1fr]    (2/3 table + 1/3 sidebar ~340px)
```

### Sidebar Structure

```
┌─────────────────────────┐
│   Currency (existing)   │
├─────────────────────────┤
│                         │
│      PAPERDOLL          │
│   (equipment slots)     │
│                         │
├─────────────────────────┤
│   Attunement (0/3)      │
│   - Item 1              │
│   - Item 2              │
├─────────────────────────┤
│   Encumbrance (existing)│
└─────────────────────────┘
```

## Paperdoll Component

### Visual Layout

SVG silhouette with slot markers positioned around the figure:

```
        [head]
        [neck]
[main]  [armor]  [off]
        [cloak]
[hands] [belt]  [ring1]
        [feet]  [ring2]
```

Fallback: Grid of slot boxes arranged in body shape if SVG approach is too complex.

### Behavior

- Display-only (not interactive for equipping)
- Click equipped item → scroll to it in item table
- Each slot shows: slot label + item name (or "Empty" in muted text)

### Equipment Slots

| Slot | Label |
|------|-------|
| head | Head |
| neck | Neck |
| cloak | Cloak |
| armor | Armor |
| belt | Belt |
| hands | Hands |
| ring_1 | Ring |
| ring_2 | Ring |
| feet | Feet |
| main_hand | Main Hand |
| off_hand | Off Hand |

## Slot Mapping Logic

### Type-to-Slot Table

| Item Type Code | Valid Slots | Auto-Equip To |
|----------------|-------------|---------------|
| M (Melee Weapon) | main_hand, off_hand | main_hand |
| R (Ranged Weapon) | main_hand | main_hand |
| ST (Staff) | main_hand | main_hand |
| RD (Rod) | main_hand | main_hand |
| WD (Wand) | main_hand | main_hand |
| LA, MA, HA (Armor) | armor | armor |
| S (Shield) | off_hand | off_hand |
| RG (Ring) | ring_1, ring_2 | ring_1 (or ring_2 if occupied) |
| W (Wondrous Item) | body slots* | **prompt user** |
| G, P, SC, A, $ | none | backpack only |

*Body slots = head, neck, cloak, belt, hands, feet

### Equip Flow

```
User clicks "Equip" on item
    ↓
Get item type from item.type or item.item_type
    ↓
Is type in known mapping?
    ├─ YES, single slot → auto-equip
    ├─ YES, multiple slots → check if default occupied
    │       ├─ Default free → auto-equip to default
    │       └─ Default occupied → show slot picker
    └─ NO (Wondrous Item) → show slot picker modal
```

## Slot Picker Modal

### When Shown

- Equipping a Wondrous Item (W)
- Equipping a Ring when choosing between ring_1/ring_2
- Equipping a weapon when main_hand is occupied

### UI Design

```
┌────────────────────────────────────┐
│  Equip: [Item Name]                │
├────────────────────────────────────┤
│  Where do you want to equip this?  │
│                                    │
│  ○ Head                            │
│  ○ Neck                            │
│  ● Cloak        ← pre-selected     │
│  ○ Belt            if name matches │
│  ○ Hands                           │
│  ○ Feet                            │
│                                    │
│  [ Cancel ]           [ Equip ]    │
└────────────────────────────────────┘
```

### Smart Pre-selection (Nice-to-have)

Pattern match item name for helpful defaults:

| Pattern | Pre-select |
|---------|------------|
| Boot, Boots | feet |
| Cloak, Cape | cloak |
| Belt, Girdle | belt |
| Helm, Helmet, Hat, Circlet | head |
| Amulet, Necklace, Periapt | neck |
| Gloves, Gauntlets, Bracers | hands |

## Component Changes

### New Components

| Component | Purpose |
|-----------|---------|
| `EquipmentPaperdoll.vue` | SVG silhouette with 11 slot positions |
| `EquipSlotPickerModal.vue` | Radio button slot selection modal |

### Modified Components

| Component | Changes |
|-----------|---------|
| `inventory.vue` | Grid 2fr/1fr, slot picker modal state, equip flow |
| `ItemTable.vue` | Update handleEquip() to emit slot or trigger picker |
| `EquipmentStatus.vue` | Replace internals with paperdoll, keep attunement section |

### New Utility

`app/utils/equipmentSlots.ts`:
- `SLOT_MAPPING` constant
- `getValidSlots(itemType: string): string[]`
- `getDefaultSlot(itemType: string): string | null`
- `needsSlotPicker(itemType: string): boolean`
- `guessSlotFromName(itemName: string): string | null`

### Unchanged

- Currency display
- Encumbrance bar
- Attunement section (content unchanged, just repositioned)
- All existing modals (AddLoot, Shop, Sell, EditQty, ItemDetail)

## Testing

### Unit Tests

- `equipmentSlots.test.ts` - Slot mapping logic, name pattern matching

### Component Tests

- `EquipmentPaperdoll.test.ts` - Renders all slots, shows equipped items, click emits
- `EquipSlotPickerModal.test.ts` - Shows valid slots, pre-selection, emits selection

### Integration Tests

- Equip armor → auto-assigns to armor slot
- Equip ring → auto-assigns to ring_1, then ring_2
- Equip wondrous item → opens slot picker → equips to selected slot

## Backend Dependency

**Issue:** #589

Backend to analyze adding `equipment_slot` field to items. This would allow:
- Accurate slot assignment for Wondrous Items
- Remove need for name pattern matching

Until then, frontend uses manual slot selection for Wondrous Items.

## Out of Scope

- Drag-and-drop equipping from table to paperdoll
- Unequip from paperdoll (use table actions)
- Visual item icons in slots (text names only for now)
- Slot conflict warnings (handled by backend validation)
