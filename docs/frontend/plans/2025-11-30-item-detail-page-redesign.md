# Item Detail Page Redesign

**Date:** 2025-11-30
**Status:** Approved for Implementation

## Overview

Redesign the item detail page with type-adaptive layouts that surface the most important information based on item category. Replaces the current flat accordion structure with specialized sections for weapons, armor, and magic items.

## Goals

1. **Character building**: Clear requirements, attunement info, comparison-friendly
2. **At-the-table reference**: Combat stats prominent, spell costs scannable
3. **DM loot planning**: Rarity visible, power level clear

## Design Decisions

### Item Categories

| Category | Item Types | Key UI Elements |
|----------|------------|-----------------|
| **Weapons** | Melee Weapon, Ranged Weapon, Ammunition | Damage, type, range, properties |
| **Armor** | Light/Medium/Heavy Armor, Shield | AC, STR req, stealth disadvantage |
| **Charged Magic** | Any with `charges_max` | Charges, recharge, spell costs |
| **Consumables** | Potion, Scroll | Effect description |
| **General** | Gear, Trade Goods, Rod, Ring, Wand, Staff, Wondrous | Description-focused |

**Key insight**: Magic overlays on base categories. Staff of the Magi shows BOTH weapon stats AND magic section.

### Page Structure

```
┌─────────────────────────────────────────────────────────────┐
│ Breadcrumb                                                  │
├─────────────────────────────────────────────────────────────┤
│ HERO: Name + Badges + Image (side-by-side)                  │
├─────────────────────────────────────────────────────────────┤
│ COMBAT STATS (if weapon or armor) - dedicated section       │
├─────────────────────────────────────────────────────────────┤
│ MAGIC SECTION (if charged/attuned) - charges, modifiers     │
├─────────────────────────────────────────────────────────────┤
│ SPELLS TABLE (if item grants spells) - grouped by cost      │
├─────────────────────────────────────────────────────────────┤
│ DESCRIPTION (always visible)                                │
├─────────────────────────────────────────────────────────────┤
│ QUICK INFO: Cost • Weight • Source                          │
├─────────────────────────────────────────────────────────────┤
│ ACCORDION: Abilities, Data Tables, Prerequisites, etc.      │
└─────────────────────────────────────────────────────────────┘
```

### Charges Display

Clean numeric format: `50 charges (recharges 4d6+2 at dawn)`

No visual progress bar - keeps it simple and informative.

## Components

### New Components (6)

| Component | Purpose |
|-----------|---------|
| `ItemHero.vue` | Name, type/rarity badges, magic indicators, image |
| `ItemWeaponStats.vue` | Damage dice, damage type, range, properties grid |
| `ItemArmorStats.vue` | AC, STR requirement, stealth disadvantage, dex modifier |
| `ItemMagicSection.vue` | Charges, attunement requirements, modifiers |
| `ItemSpellsTable.vue` | Spells grouped by charge cost |
| `ItemQuickInfo.vue` | Cost, weight, source in compact row |

### New Composable (1)

**`useItemDetail.ts`**
```typescript
interface UseItemDetailReturn {
  item: ComputedRef<Item | null>
  loading: ComputedRef<boolean>
  error: ComputedRef<Error | null>

  // Type detection
  isWeapon: ComputedRef<boolean>
  isArmor: ComputedRef<boolean>
  isShield: ComputedRef<boolean>
  isCharged: ComputedRef<boolean>
  isMagic: ComputedRef<boolean>
  hasSpells: ComputedRef<boolean>

  // Derived data
  attunementText: ComputedRef<string | null>
  spellsByChargeCost: ComputedRef<Map<string, ItemSpell[]>>
  damageDisplay: ComputedRef<string | null>
  acDisplay: ComputedRef<string | null>
  costDisplay: ComputedRef<string | null>
  categoryColor: ComputedRef<string>
}
```

### Reused Components (6)

- `UiDetailBreadcrumb`
- `UiDetailDescriptionCard`
- `UiSourceDisplay`
- `UiModifiersDisplay`
- `UiAccordionAbilitiesList`
- `UiAccordionRandomTablesList`

## Example Renders

### Simple Weapon (Longsword)
- Hero: Name + [Melee Weapon] [Common]
- Weapon Stats: 1d8 Slashing (1d10 versatile), [Versatile] [Martial]
- Description
- Quick Info: 15 gp • 3 lb • PHB p.149

### Armor (Plate)
- Hero: Name + [Heavy Armor] [Uncommon]
- Armor Stats: AC 18, STR 15, Stealth Disadvantage, No Dex
- Warning: Speed -10 if STR < 15
- Description
- Quick Info: 1,500 gp • 65 lb • PHB p.145

### Complex Magic (Staff of the Magi)
- Hero: Name + [Staff] [Legendary] [+2] [Attunement]
- Weapon Stats: 1d6 Bludgeoning (1d8 versatile), [Versatile]
- Magic Section: 50 charges, attunement, +2 bonuses
- Spells Table: Grouped by cost (Free, 2, 3, 4, 5, 7 charges)
- Description
- Quick Info: 4 lb • DMG p.203
- Accordion: Abilities, Saving Throws

## Testing Strategy

- Unit tests for `useItemDetail` composable (type detection, computed values)
- Component tests for each new component with mock data
- Integration tests for page with different item types
- Snapshot tests for layout consistency

## Out of Scope

- Interactive charge tracking (future feature)
- Item comparison view
- Related items suggestions
