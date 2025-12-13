# Inventory Tab Design

**Issue:** #555 - Character Sheet: Inventory Tab
**Epic:** #552 - Character Sheet Tabbed Interface Enhancement
**Created:** 2025-12-13

## Overview

Full inventory management tab for the character sheet with item grouping, equipment slots, purchasing, and optional encumbrance tracking. This establishes the foundation for the top-level tab system described in epic #552.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Tab routing | URL path segments (`/characters/:id/inventory`) | Shareable, bookmarkable, feels like pages |
| Layout | Two-column (item list + sticky sidebar) | Desktop-friendly, stacks on mobile |
| Item grouping | By type, equipped items highlighted | Best of both organizational approaches |
| Equipment slots | Attunement (3) + Wielded (main/off hand) | D&D RAW, no invented slot system |
| Slot selection | Dropdown with eligible items | Quick, minimal UI |
| Shop feature | Add Item modal with purchase flow | Self-contained, integrates with currency |
| Encumbrance | Optional toggle, off by default | Many tables don't track weight |

## Architecture

### Routing

```
/characters/[publicId]/           → Default view (current sheet)
/characters/[publicId]/inventory  → Inventory tab
/characters/[publicId]/battle     → Battle tab (future)
/characters/[publicId]/spells     → Spells tab (future)
```

### File Structure

```
app/pages/characters/[publicId]/
├── index.vue              # Default tab (existing, minimal changes)
├── inventory.vue          # New inventory tab page

app/components/character/
├── TabNavigation.vue      # Shared tab bar component
└── inventory/
    ├── InventoryLayout.vue      # Two-column layout wrapper
    ├── ItemList.vue             # Grouped item list (left column)
    ├── ItemGroup.vue            # Single type group with items
    ├── EquipmentSlots.vue       # Attunement + wielded slots (sidebar)
    ├── EncumbranceBar.vue       # Weight tracking (optional)
    └── AddItemModal.vue         # Search & add items with purchase
```

### Composables

- `useEquipmentSlots()` - Manages slot assignments, equip/unequip API calls
- `useItemGrouping()` - Groups items by type, filters by equipped status

## Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  [Tab Navigation: Default | Inventory | Battle | Spells | ...]  │
├────────────────────────────────────┬────────────────────────────┤
│                                    │                            │
│  ITEM LIST (scrollable)            │  SIDEBAR (sticky)          │
│                                    │                            │
│  ┌─ Weapons ─────────────────┐     │  ┌─ Equipment Slots ────┐  │
│  │ ⚔ Longsword [equipped]    │     │  │ Wielded:             │  │
│  │ ⚔ Shortbow                │     │  │  [Main Hand ▾]       │  │
│  │ 🗡 Dagger (×2)            │     │  │  [Off Hand ▾]        │  │
│  └───────────────────────────┘     │  │                      │  │
│                                    │  │ Attunement (0/3):    │  │
│  ┌─ Armor ───────────────────┐     │  │  [Empty ▾]           │  │
│  │ 🛡 Chain Mail [equipped]  │     │  │  [Empty ▾]           │  │
│  │ 🛡 Shield [equipped]      │     │  │  [Empty ▾]           │  │
│  └───────────────────────────┘     │  └──────────────────────┘  │
│                                    │                            │
│  ┌─ Adventuring Gear ────────┐     │  ┌─ Currency ───────────┐  │
│  │ 🎒 Backpack               │     │  │  PP: 0    GP: 15     │  │
│  │ 🔦 Torch (×5)             │     │  │  EP: 0    SP: 30     │  │
│  └───────────────────────────┘     │  │      CP: 50          │  │
│                                    │  └──────────────────────┘  │
│                                    │                            │
│         [+ Add Item]               │  ┌─ Encumbrance ────────┐  │
│                                    │  │  45 / 150 lbs [━━━░] │  │
│                                    │  └──────────────────────┘  │
└────────────────────────────────────┴────────────────────────────┘
```

### Responsive Behavior

- **Desktop (lg+)**: Two-column as shown, sidebar sticky on scroll
- **Mobile/Tablet**: Single column - sidebar panels stack above item list

### Visual Treatment for Equipped Items

- Highlighted background (subtle primary color tint)
- "[equipped]" badge
- Check icon instead of default item type icon

## Equipment Slots

### Slot Structure

1. **Wielded** (what you're holding)
   - Main Hand - Weapons, shields, or items like torches
   - Off Hand - Shield, light weapon (dual-wield), or two-handed weapon indicator

2. **Attunement** (3 slots per D&D rules)
   - Any magic item requiring attunement
   - Shows item name + "Attuned" indicator
   - Counter shows "Attunement (2/3)"

### Slot Inference

Slots are inferred from item type - no backend changes needed:
- Armor category → Body (implicit, not a slot)
- Shield → Off Hand
- Weapon (two-handed) → Main Hand (blocks Off Hand)
- Weapon (light) → Main Hand or Off Hand
- Weapon (other) → Main Hand
- Magic item (requires attunement) → Attunement slot

### Equip/Unequip Flow

1. Player clicks slot dropdown
2. Dropdown shows eligible items filtered by slot type
3. Select item → `PATCH /characters/:id/equipment/:equipmentId` with `{ equipped: true }`
4. Previous item in slot auto-unequips
5. Optimistic UI update, rollback on error

## Add Item Modal (Purchase Flow)

```
┌─────────────────────────────────────────────────────┐
│  Add Item                                    [×]    │
├─────────────────────────────────────────────────────┤
│  🔍 [Search items...                          ]     │
│                                                     │
│  Filter: [All ▾] [Weapons ▾] [Armor ▾] [Gear ▾]    │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │ Longsword                      15 gp    1d8 ⚔ │ │
│  │ Martial weapon, versatile                     │ │
│  ├───────────────────────────────────────────────┤ │
│  │ Longbow                        50 gp    1d8 🏹 │ │
│  └───────────────────────────────────────────────┘ │
│                                                     │
│  ┌─ Selected ───────────────────────────────────┐  │
│  │ Longsword                         Qty: [1]   │  │
│  │                                              │  │
│  │ Base Price:    15 gp                         │  │
│  │ Actual Cost:   [15] gp  ← editable           │  │
│  │                                              │  │
│  │ ☐ Free (gift/loot/found)                     │  │
│  └──────────────────────────────────────────────┘  │
│                                                     │
│  Your Gold: 45 gp    After: 30 gp                  │
│                                                     │
│             [Cancel]  [Purchase & Add]              │
└─────────────────────────────────────────────────────┘
```

### Purchase Flow

1. Select item → shows base price from item data
2. Editable "Actual Cost" field - defaults to base price
3. "Free" checkbox - sets cost to 0 (for loot/gifts)
4. Shows current gold and "After" balance preview
5. "Purchase & Add" does two API calls:
   - `PATCH /characters/:id/currency` with negative delta
   - `POST /characters/:id/equipment` with item
6. If insufficient funds → show warning, disable button (unless "Free" checked)

### Custom Items

"Custom Item" option at bottom of search results for items not in compendium. Opens inline fields for `custom_name` and `custom_description`.

## Encumbrance

### Panel (when enabled)

```
┌─ Encumbrance ──────────────────────────┐
│                                        │
│  45 / 150 lbs                          │
│  [━━━━━━━━━━━━░░░░░░░░░░░░░░░░░░░░░━] │
│                                        │
│  Carrying Capacity: 150 lbs            │
│  Push/Drag/Lift: 300 lbs               │
│                                        │
│  [✓] Track encumbrance                 │
└────────────────────────────────────────┘
```

### Visual States

| Weight % | Color | Label |
|----------|-------|-------|
| 0-66% | Green | Normal |
| 67-99% | Yellow/Warning | Approaching limit |
| 100%+ | Red/Error | Over capacity |

### Toggle Persistence

Saved to localStorage: `encumbrance-tracking-{publicId}: true/false`

## Error Handling

| Action | Error | User Feedback |
|--------|-------|---------------|
| Equip item | API fails | Toast error, rollback optimistic update |
| Purchase item | Insufficient funds | Inline warning, disable button |
| Purchase item | API fails | Toast error, no currency deducted |
| Add custom item | Validation fails | Inline field errors |
| Load inventory | Fetch fails | Error alert with retry button |

## Play Mode Integration

All interactive features require play mode:
- Dropdowns disabled in view mode
- "Add Item" button hidden
- Currency displays but isn't editable

## Test Coverage

| Component | Test Focus |
|-----------|------------|
| `TabNavigation.vue` | Active tab highlighting, route generation |
| `ItemList.vue` | Grouping by type, equipped highlighting, empty state |
| `ItemGroup.vue` | Collapsible groups, item rendering |
| `EquipmentSlots.vue` | Dropdown filtering, equip/unequip calls, slot inference |
| `EncumbranceBar.vue` | Weight calculation, color thresholds, toggle persistence |
| `AddItemModal.vue` | Search, quantity, cost editing, purchase flow, validation |
| Integration | Full purchase flow, equip from inventory, API error handling |
