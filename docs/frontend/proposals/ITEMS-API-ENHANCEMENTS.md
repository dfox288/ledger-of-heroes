# Items API Enhancement Proposals

**Date:** 2025-11-26
**Status:** Proposal
**API Endpoint:** `/api/v1/items`
**Overall Assessment:** 🟢 PASS - Excellent data structure with comprehensive item modeling

---

## Executive Summary

The Items API is **production-ready** with excellent data modeling for weapons, armor, and magic items. The API properly handles D&D 5e mechanics like versatile weapons, armor dexterity modifiers, attunement, and magic item charges.

### Current Strengths
- Comprehensive weapon data (damage dice, damage types, versatile, range)
- Excellent armor modeling (AC, dex modifiers, stealth disadvantage, strength requirements)
- Magic item support (rarity, attunement, charges, recharge timing)
- Property system for weapon properties (Finesse, Heavy, Two-Handed, etc.)
- Modifier system for AC calculations and skill effects
- Abilities system for magic item rolls and effects
- Proper cost modeling in copper pieces (allows precise GP/SP/CP calculations)

### Minor Issues Found
- No `/item-types` endpoint (404)
- Some minor data inconsistencies in filter field names

---

## Logical Correctness Analysis ✅

### Weapon Verification (PHB p.146-149)

| Weapon | Damage | Type | Properties | API Status |
|--------|--------|------|------------|------------|
| Longsword | 1d8/1d10 | Slashing | Versatile, Martial | ✅ Correct |
| Dagger | 1d4 | Piercing | Finesse, Light, Thrown (20/60) | ✅ Correct |
| Longbow | 1d8 | Piercing | Ammunition, Heavy, Two-Handed, Range 150/600 | ✅ Correct |

**D&D Rules Validation:**
- ✅ Longsword: 1d8 slashing, versatile 1d10 (PHB p.149) - **CORRECT**
- ✅ Dagger: 1d4 piercing, finesse, light, thrown 20/60 (PHB p.149) - **CORRECT**
- ✅ Longbow: 1d8 piercing, ammunition, heavy, two-handed, 150/600 (PHB p.149) - **CORRECT**

### Armor Verification (PHB p.144-145)

| Armor | Base AC | Type | Dex Mod | Stealth | STR Req | API Status |
|-------|---------|------|---------|---------|---------|------------|
| Studded Leather | 12 | Light | Full | No | None | ✅ Correct |
| Half Plate | 15 | Medium | Max +2 | Disadvantage | None | ✅ Correct |
| Chain Mail | 16 | Heavy | None | Disadvantage | 13 | ✅ Correct |
| Plate | 18 | Heavy | None | Disadvantage | 15 | ✅ Correct |
| Shield | +2 | Shield | N/A | No | None | ✅ Correct |

**D&D Rules Validation:**
- ✅ Plate Armor: AC 18, STR 15 required, stealth disadvantage (PHB p.145) - **CORRECT**
- ✅ Half Plate: AC 15 + Dex (max 2), stealth disadvantage (PHB p.145) - **CORRECT**
- ✅ Studded Leather: AC 12 + Dex (PHB p.144) - **CORRECT**

### Magic Item Verification (DMG)

| Item | Rarity | Attunement | API Status |
|------|--------|------------|------------|
| Bag of Holding | Uncommon | No | ✅ Correct |
| Flame Tongue | Rare | Yes | ✅ Correct |
| Apparatus of Kwalish | Legendary | No | ✅ Correct |
| Armor of Invulnerability | Legendary | Yes | ✅ Correct |

---

## Structural Soundness Analysis ✅

### What Works Excellently

1. **Weapon Property System**
   ```json
   {
     "properties": [
       { "id": 2, "code": "F", "name": "Finesse", "description": "..." },
       { "id": 4, "code": "L", "name": "Light", "description": "..." },
       { "id": 7, "code": "T", "name": "Thrown", "description": "..." }
     ]
   }
   ```
   - Complete property codes match official shorthand
   - Descriptions explain mechanics

2. **Armor Modifier System**
   ```json
   {
     "modifiers": [
       {
         "modifier_category": "ac_base",
         "value": "15",
         "condition": "dex_modifier: max_2"
       },
       {
         "modifier_category": "skill",
         "skill": { "name": "Stealth" },
         "value": "disadvantage"
       },
       {
         "modifier_category": "speed",
         "value": "-10",
         "condition": "strength < 15"
       }
     ]
   }
   ```
   - Handles complex AC calculations
   - Supports conditional modifiers
   - Properly models stealth disadvantage

3. **Magic Item Abilities**
   ```json
   {
     "abilities": [
       {
         "ability_type": "roll",
         "name": "2d6",
         "roll_formula": "2d6"
       }
     ]
   }
   ```
   - Captures extra damage dice
   - Supports spell-like abilities

4. **Item Type Classification**
   ```json
   {
     "item_type": {
       "id": 2,
       "code": "M",
       "name": "Melee Weapon"
     }
   }
   ```

   **Observed Item Types:**
   | Code | Name | Description |
   |------|------|-------------|
   | M | Melee Weapon | Close combat weapons |
   | R | Ranged Weapon | Projectile weapons |
   | LA | Light Armor | Full DEX bonus |
   | MA | Medium Armor | Max +2 DEX bonus |
   | HA | Heavy Armor | No DEX bonus |
   | S | Shield | AC bonus items |
   | G | Adventuring Gear | General equipment |
   | W | Wondrous Item | Miscellaneous magic |
   | $ | Trade Goods | Gems, art objects |

5. **Cost in Copper Pieces**
   ```json
   {
     "cost_cp": 1500  // 15 GP for longsword
   }
   ```
   - Allows precise calculations (1 GP = 100 CP)
   - Frontend can display in GP/SP/CP as needed

---

## Feature Completeness Analysis ✅

### Essential Fields Present

| Field | Present | Notes |
|-------|---------|-------|
| name, slug | ✅ | All items |
| item_type | ✅ | Nested object with code/name |
| rarity | ✅ | common, uncommon, rare, very rare, legendary |
| requires_attunement | ✅ | Boolean |
| is_magic | ✅ | Boolean |
| cost_cp | ✅ | Copper pieces (null for priceless items) |
| weight | ✅ | String with decimal |
| **Weapons** | | |
| damage_dice | ✅ | "1d8", "2d6", etc. |
| versatile_damage | ✅ | "1d10" when applicable |
| damage_type | ✅ | Nested object with code/name |
| range_normal | ✅ | Integer (feet) |
| range_long | ✅ | Integer (feet) |
| properties | ✅ | Array of property objects |
| **Armor** | | |
| armor_class | ✅ | Base AC value |
| strength_requirement | ✅ | Minimum STR or null |
| stealth_disadvantage | ✅ | Boolean |
| modifiers | ✅ | AC calculation, skill effects |
| **Magic Items** | | |
| charges_max | ✅ | Maximum charges |
| recharge_formula | ✅ | Dice formula for recharge |
| recharge_timing | ✅ | "dawn", "dusk", etc. |
| abilities | ✅ | Spells, rolls, special abilities |
| spells | ✅ | Array of spell references |
| prerequisites | ✅ | Attunement requirements |

---

## Medium Priority Enhancements

### 1. ~~Add `/item-types` Reference Endpoint~~ ✅ EXISTS

**Status:** ✅ Already exists at `/api/v1/lookups/item-types`

**Endpoint:** `GET /api/v1/lookups/item-types`

**Response (16 item types):**
```json
{
  "data": [
    { "id": 1, "code": "A", "name": "Ammunition", "description": "Arrows, bolts, sling bullets, and other projectiles" },
    { "id": 2, "code": "M", "name": "Melee Weapon", "description": "Weapons used for close combat" },
    { "id": 3, "code": "R", "name": "Ranged Weapon", "description": "Weapons used for ranged combat" },
    ...
  ]
}
```

**Note:** The endpoint is under `/lookups/` prefix, not root `/api/v1/`.

**Benefit:** Frontend can build filter dropdowns without hardcoding values. ✅

---

### 2. Add `proficiency_category` Field

**Current State:** Proficiency is in description text or properties array.

**Proposed Enhancement:**
```json
{
  "proficiency_category": "martial_melee",  // or "simple_melee", "simple_ranged", etc.
  "proficiency_specific": "longsword"
}
```

**D&D Context:**
- Simple vs Martial proficiency affects character class choices
- Specific weapon proficiency (Elf → Longsword) needs separate tracking

**Benefit:** Enables "Show weapons I'm proficient with" filter for character builders.

---

### 3. Add `price_gp` Computed Field

**Current State:** Only `cost_cp` available.

**Proposed Enhancement:**
```json
{
  "cost_cp": 1500,
  "price_gp": 15,
  "price_display": "15 gp"
}
```

**Benefit:** Reduces frontend computation, improves display consistency.

---

### 4. ~~Fix `type_code` Filter~~ ✅ RESOLVED

**Status:** ✅ Fixed 2025-11-26 - Was stale Meilisearch index

**Field Name:** `type_code`

**Now Working:**
```bash
curl "http://localhost:8080/api/v1/items?filter=type_code=M&per_page=5"
# Response: { "meta": { "total": 482 }, "data": [5 items] } ✅

curl "http://localhost:8080/api/v1/items?filter=type_code=R&per_page=5"
# Response: { "meta": { "total": 138 }, "data": [5 items] } ✅

curl "http://localhost:8080/api/v1/items?filter=type_code=LA&per_page=3"
# Response: { "meta": { "total": 115 }, "data": [3 items] } ✅
```

**Item Type Codes (from `/api/v1/lookups/item-types`):**
| Code | Name |
|------|------|
| M | Melee Weapon |
| R | Ranged Weapon |
| LA | Light Armor |
| MA | Medium Armor |
| HA | Heavy Armor |
| A | Ammunition |
| ... | (16 total) |

**Benefit:** Equipment shop views and category filtering now enabled. ✅

---

## Low Priority / Nice-to-Have

### 5. Add `weight_lb` Numeric Field

**Current State:** `weight: "3.00"` (string)

**Proposed Enhancement:**
```json
{
  "weight": "3.00",
  "weight_lb": 3.0
}
```

**Benefit:** Enables numeric comparison for encumbrance calculations.

---

### 6. Add `magic_bonus` Field for +X Items

**Current State:** Bonus is in item name ("Longsword +1")

**Proposed Enhancement:**
```json
{
  "name": "Longsword +1",
  "magic_bonus": 1,
  "base_item_slug": "longsword"
}
```

**Benefit:**
- Filter by enhancement bonus
- Link magic items to base items
- Calculate total attack/damage bonuses

---

### 7. Add `attunement_requirements` Structured Field

**Current State:** Requirements in `prerequisites` array or description.

**Proposed Enhancement:**
```json
{
  "attunement_requirements": {
    "required": true,
    "class": ["cleric", "paladin"],
    "alignment": null,
    "other": null
  }
}
```

**D&D Context:**
- Holy Avenger: Requires attunement by a paladin
- Robes of the Archmagi: Requires attunement by a sorcerer, warlock, or wizard

---

### 8. Add `random_table_refs` for DMG Treasure Tables

**Current State:** Some items mention "Found on: Magic Item Table A"

**Proposed Enhancement:**
```json
{
  "random_table_refs": ["magic_item_table_a", "magic_item_table_b"]
}
```

**Benefit:** Enables treasure generation tools.

---

## Implementation Priority Matrix

| Enhancement | Effort | Impact | Priority |
|-------------|--------|--------|----------|
| ~~Add `/item-types` endpoint~~ | ~~Low~~ | ~~High~~ | ✅ **EXISTS** at `/lookups/item-types` |
| Add `proficiency_category` | Medium | High | 🟡 Medium |
| Add `price_gp` computed field | Low | Medium | 🟡 Medium |
| ~~Fix `type_code` filter~~ | ~~Low~~ | ~~High~~ | ✅ **DONE** - was stale index |
| Add `weight_lb` numeric | Low | Low | 🟢 Low |
| Add `magic_bonus` field | Medium | Medium | 🟢 Low |
| Add `attunement_requirements` | Medium | Low | 🟢 Low |
| Add `random_table_refs` | Low | Low | 🟢 Low |

---

## Verification Checklist

The following was verified against official D&D 5e sources:

**Weapons (PHB p.146-149):**
- [x] Longsword: 1d8 slashing, versatile 1d10, martial
- [x] Dagger: 1d4 piercing, finesse, light, thrown 20/60, simple
- [x] Longbow: 1d8 piercing, ammunition, heavy, two-handed, 150/600, martial

**Armor (PHB p.144-145):**
- [x] Plate: AC 18, STR 15, stealth disadvantage
- [x] Half Plate: AC 15 + Dex (max 2), stealth disadvantage
- [x] Chain Mail: AC 16, STR 13, stealth disadvantage
- [x] Studded Leather: AC 12 + Dex (full)
- [x] Shield: +2 AC

**Magic Items (DMG):**
- [x] Bag of Holding: Uncommon, no attunement (DMG p.153)
- [x] Flame Tongue Longsword: Rare, requires attunement, +2d6 fire (DMG p.170)

---

## Summary

The Items API is **production-ready** with comprehensive D&D 5e data modeling. The weapon and armor systems correctly handle all mechanical details including:

- Damage dice and types
- Versatile weapons
- Ranged weapon distances
- Weapon properties (Finesse, Heavy, Two-Handed, etc.)
- Armor AC with dexterity modifier conditions
- Stealth disadvantage
- Strength requirements
- Magic item rarities and attunement
- Charge-based items with recharge timing

The suggested enhancements are quality-of-life improvements for frontend filtering and display, not corrections to data accuracy.

---

## Related Documentation

- Frontend item pages: `app/pages/items/`
- Item filter store: `app/stores/itemFilters.ts`
- Backend API: `/Users/dfox/Development/dnd/importer`
