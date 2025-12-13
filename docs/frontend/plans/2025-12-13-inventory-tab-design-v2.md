# Inventory Tab Design v2 (Refined)

**Issue:** #555 - Character Sheet: Inventory Tab
**Epic:** #552 - Character Sheet Tabbed Interface Enhancement
**Created:** 2025-12-13
**Refined:** 2025-12-13 (brainstorming session)

## Overview

Full inventory management page for the character sheet with item actions, equipment status sidebar, purchasing, and optional encumbrance tracking. URL-routed for shareability.

## Key Changes from v1

| Aspect | v1 | v2 (Refined) |
|--------|-----|--------------|
| Equipping UI | Sidebar dropdowns | Item-centric actions (click item → menu) |
| Sidebar role | Interactive equip | Read-only status display |
| Add item | One overloaded modal | Two entry points: Add Loot / Shop |
| Item grouping | Frontend sorts | Backend returns pre-sorted |
| Equipped items | Separate section | Inline badges, sidebar shows status |
| Item detail | Not specified | Expand inline |
| Drop vs Sell | Not specified | Separate actions |

## Architecture

### Routing

```
/characters/[publicId]/              → Main character sheet
/characters/[publicId]/inventory     → Inventory management (NEW)
/characters/[publicId]/spells        → Spells tab (future)
/characters/[publicId]/battle        → Battle tab (future)
```

### File Structure

```
app/pages/characters/[publicId]/
├── index.vue                    # Main sheet (remove Equipment tab)
├── inventory.vue                # NEW: Full inventory page

app/components/character/
├── TabNavigation.vue            # NEW: Shared nav bar across pages
└── inventory/
    ├── InventoryLayout.vue      # Two-column layout wrapper
    ├── ItemList.vue             # Searchable item list
    ├── ItemRow.vue              # Single item with expand/actions
    ├── ItemActions.vue          # Action menu (equip, sell, drop)
    ├── EquipmentStatus.vue      # Sidebar: wielded/armor/attuned
    ├── EncumbranceBar.vue       # Optional weight tracking
    ├── AddLootModal.vue         # Quick add (free items)
    └── ShopModal.vue            # Purchase flow with currency
```

### Composables

- `useEquipmentSlots()` - Manages slot assignments, equip/unequip API calls
- `useInventoryActions()` - Item actions (sell, drop, consume, edit qty)

## Layout

```
┌─────────────────────────────────────────────────────────────────┐
│  [Tab Navigation: Overview | Inventory | Spells | Battle]       │
├────────────────────────────────────┬────────────────────────────┤
│                                    │                            │
│  🔍 [Search items...]              │  SIDEBAR (sticky)          │
│                                    │                            │
│  ITEM LIST (scrollable)            │  ┌─ Wielded ───────────┐  │
│  (backend returns pre-sorted:      │  │ Main: Longsword      │  │
│   equipped first, then by type)    │  │ Off: Shield          │  │
│                                    │  └─────────────────────┘  │
│  ┌─ Weapons ────────────────────┐  │                            │
│  │ ⚔ Longsword    [Main Hand]   │  │  ┌─ Armor ─────────────┐  │
│  │ 🛡 Shield      [Off Hand]    │  │  │ Chain Mail (AC 16)   │  │
│  │ 🏹 Shortbow                  │  │  └─────────────────────┘  │
│  │ 🗡 Dagger (×2)               │  │                            │
│  └──────────────────────────────┘  │  ┌─ Attuned (1/3) ─────┐  │
│                                    │  │ Ring of Protection   │  │
│  ┌─ Armor ──────────────────────┐  │  │ [Empty]              │  │
│  │ 🧥 Chain Mail  [Worn]        │  │  │ [Empty]              │  │
│  └──────────────────────────────┘  │  └─────────────────────┘  │
│                                    │                            │
│  ┌─ Adventuring Gear ───────────┐  │  ┌─ Currency ──────────┐  │
│  │ 🎒 Backpack                  │  │  │ (existing card)      │  │
│  │ 🔦 Torch (×5)                │  │  └─────────────────────┘  │
│  └──────────────────────────────┘  │                            │
│                                    │  ┌─ Encumbrance ───────┐  │
│  [+ Add Loot]  [🛒 Shop]           │  │ 45/150 lbs [━━━░░░] │  │
│                                    │  │ [✓] Track weight     │  │
│                                    │  └─────────────────────┘  │
└────────────────────────────────────┴────────────────────────────┘
```

### Responsive Behavior

- **Desktop (lg+)**: Two-column, sidebar sticky on scroll
- **Mobile/Tablet**: Sidebar panels stack ABOVE item list

### Sidebar Sections (Read-Only Status)

1. **Wielded**: Main Hand / Off Hand - click jumps to item in list
2. **Armor**: Current worn armor with AC displayed
3. **Attuned (X/3)**: Up to 3 items, empty slots shown
4. **Currency**: Existing `CurrencyCard` component
5. **Encumbrance**: Progress bar + toggle (localStorage persisted)

## Item Interactions

### Item Row (Collapsed)

```
┌─────────────────────────────────────────────────────────────┐
│ ⚔ Longsword                        [Main Hand]  ×1    [⋮]  │
└─────────────────────────────────────────────────────────────┘
```

### Item Row (Expanded - click to toggle)

```
┌─────────────────────────────────────────────────────────────┐
│ ⚔ Longsword                        [Main Hand]  ×1    [⋮]  │
├─────────────────────────────────────────────────────────────┤
│ Martial weapon, versatile                                   │
│ Damage: 1d8 slashing (1d10 two-handed)                     │
│ Weight: 3 lbs                                               │
│                                                             │
│ [Unequip]  [Edit Qty]  [Sell]  [Drop]                      │
└─────────────────────────────────────────────────────────────┘
```

### Equipped Badges

Items show inline badges: `[Main Hand]` `[Off Hand]` `[Worn]` `[Attuned]`

### Action Menu (Context-Aware)

| Item Type | Available Actions |
|-----------|-------------------|
| **Weapon** | Equip Main Hand, Equip Off Hand*, Unequip, Edit Qty, Sell, Drop |
| **Armor** | Equip (auto-swaps), Unequip, Sell, Drop |
| **Shield** | Equip Off Hand, Unequip, Sell, Drop |
| **Magic (attuneable)** | Attune, Unattune, Sell, Drop |
| **Consumable** | Use (removes item), Edit Qty, Drop |
| **Other** | Edit Qty, Sell, Drop |

*Off Hand only for light weapons or if Main Hand is empty

## Equipment Slot Rules

### Slot Types & Enforcement

| Slot | Max Items | Enforcement Rule |
|------|-----------|------------------|
| **Main Hand** | 1 | Equip new → auto-unequip old |
| **Off Hand** | 1 | Equip new → auto-unequip old |
| **Armor** | 1 | Equip new → auto-unequip old (implicit slot) |
| **Attunement** | 3 | Block with message: "Unattune an item first" |

### Weapon Slot Logic

| Weapon Type | Main Hand | Off Hand | Notes |
|-------------|-----------|----------|-------|
| **Two-handed** | ✅ | ❌ Blocked | Also clears off-hand if occupied |
| **Light** | ✅ | ✅ | Can dual-wield two light weapons |
| **Versatile** | ✅ | ❌ | One-handed only (grip toggle future) |
| **Other (1H)** | ✅ | ❌ | Standard one-handed weapons |
| **Shield** | ❌ | ✅ | Always off-hand |

### Two-Handed Special Case

When equipping a two-handed weapon:
1. Goes to Main Hand
2. If Off Hand has item → auto-unequip to inventory
3. Off Hand shows "— (two-handed)" indicator

## Add Item Flows

### Two Entry Points

- **"+ Add Loot"** - Free items (found, looted, gifted)
- **"🛒 Shop"** - Purchase with currency deduction

### Add Loot Modal (Simple)

```
┌─────────────────────────────────────────────────┐
│  Add Loot                                  [×]  │
├─────────────────────────────────────────────────┤
│  🔍 [Search items...]                           │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ Longsword                    ⚔ Martial  │   │
│  │ Potion of Healing            🧪 Potion  │   │
│  │ Rope, Hempen (50 ft)         🎒 Gear    │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  Selected: Longsword              Qty: [1]     │
│                                                 │
│  ☐ Custom item (not in compendium)             │
│                                                 │
│                        [Cancel]  [Add to Bag]  │
└─────────────────────────────────────────────────┘
```

### Shop Modal (Full Purchase Flow)

```
┌─────────────────────────────────────────────────┐
│  Shop                                      [×]  │
├─────────────────────────────────────────────────┤
│  🔍 [Search...]   [Weapons▾] [Armor▾] [Gear▾]  │
│                                                 │
│  ┌─────────────────────────────────────────┐   │
│  │ Longsword              15 gp     ⚔ 1d8  │   │
│  │ Chain Mail            75 gp     🛡 AC16 │   │
│  │ Torch                  1 cp     🔦       │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│  ┌─ Purchase ──────────────────────────────┐   │
│  │ Longsword                    Qty: [1]   │   │
│  │                                         │   │
│  │ Base Price:     15 gp                   │   │
│  │ Your Price:     [15] gp  ← editable     │   │
│  │                                         │   │
│  │ Your Gold: 45 gp → 30 gp after          │   │
│  └─────────────────────────────────────────┘   │
│                                                 │
│                      [Cancel]  [Purchase]       │
└─────────────────────────────────────────────────┘
```

### Sell Item Flow

1. Click [⋮] → "Sell"
2. Mini-modal opens with editable price (defaults to 50% base value)
3. Confirm → item removed, currency increased
4. Toast: "Sold Longsword for 7 gp"

## Encumbrance

### Panel (when enabled)

```
┌─ Encumbrance ──────────────────────────┐
│  45 / 150 lbs                          │
│  [━━━━━━━━━━━━░░░░░░░░░░░░░░░░░░░░━]  │
│                                        │
│  [✓] Track encumbrance                 │
└────────────────────────────────────────┘
```

### Visual States

| Weight % | Color | Notes |
|----------|-------|-------|
| 0-66% | Green | Normal |
| 67-99% | Yellow | Approaching limit |
| 100%+ | Red | Over capacity (visual only, no blocking) |

### Toggle Persistence

Saved to localStorage: `encumbrance-tracking-{publicId}: true/false`

## Play Mode Integration

| Element | View Mode | Play Mode |
|---------|-----------|-----------|
| Item action menu [⋮] | Hidden | Visible |
| "+ Add Loot" button | Hidden | Visible |
| "🛒 Shop" button | Hidden | Visible |
| Currency card | Display only | Clickable to edit |
| Encumbrance toggle | Works | Works |
| Sidebar slots | Display only | Display only |

## Error Handling

| Action | Error | User Feedback |
|--------|-------|---------------|
| Equip item | API fails | Toast error, rollback optimistic update |
| Purchase item | Insufficient funds | Inline warning in modal, disable button |
| Purchase item | API fails | Toast error, currency not deducted |
| Add loot | API fails | Toast error, item not added |
| Sell item | API fails | Toast error, item kept, no gold added |
| Attune item | Already 3 attuned | Toast: "Unattune an item first" |

## Backend Requirements

To be coordinated with backend team:

1. **`location` field support**: `main_hand`, `off_hand`, `worn`, `attuned`, `inventory`
2. **Pre-sorted response**: Items returned equipped first, then grouped by type
3. **Slot inference**: Determine valid slots from item type/properties
4. **Attunement enforcement**: Max 3 attuned items per character
5. **Armor swap**: Auto-unequip previous armor when equipping new

## Test Coverage

### Component Tests

| Component | Test Focus |
|-----------|------------|
| `TabNavigation.vue` | Active tab highlighting, route generation |
| `ItemList.vue` | Search filtering, renders in backend order, empty state |
| `ItemRow.vue` | Expand/collapse, equipped badge, quantity |
| `ItemActions.vue` | Context-aware menu by item type |
| `EquipmentStatus.vue` | Displays wielded/armor/attuned |
| `EncumbranceBar.vue` | Weight calculation, color thresholds, toggle |
| `AddLootModal.vue` | Search, select, quantity, custom item |
| `ShopModal.vue` | Search, filters, price editing, purchase |

### Integration Tests

- Full purchase flow with currency deduction
- Equip flow with sidebar update and auto-unequip
- Sell flow with currency increase
- Attunement limit enforcement

### E2E (Playwright)

- Complete inventory management journey
- Mobile responsive behavior
