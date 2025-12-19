# Plan: Issue #757 - Enhanced Equipment Display

**Issue:** https://github.com/dfox288/ledger-of-heroes/issues/757
**Created:** 2025-12-19
**Status:** Ready for implementation

## Summary

Enhance ItemDetailModal to display new weapon, armor, and magic item fields from the backend CharacterEquipmentResource enhancements. Keep ItemTable minimal (no inline changes).

## Scope

**In scope:**
- Update ItemDetailModal to show weapon details (damage type, properties, range, versatile)
- Update ItemDetailModal to show armor details (armor type, DEX cap, stealth, STR req)
- Update ItemDetailModal to show magic item details (magic bonus, rarity)
- Add charge tracking UI (interactive when backend ready, static fallback otherwise)

**Out of scope:**
- ItemTable inline stats (keeping rows minimal)
- Backend changes (already done per handoff)

## Data Structure

### New fields from equipment response (in `item` object)

```typescript
// Weapon fields
damage_type: string | null        // "Slashing", "Piercing", "Bludgeoning"
properties: Array<{ id: number, code: string, name: string, description?: string }>
range: { normal: number, long: number } | null
versatile_damage: string | null   // "1d10"

// Armor fields
armor_type: 'light' | 'medium' | 'heavy' | null
max_dex_bonus: number | null      // null=unlimited, 2=medium, 0=heavy
stealth_disadvantage: boolean | null
strength_requirement: number | null

// Magic item fields
is_magic: boolean
rarity: string | null             // "common", "uncommon", "rare", etc.
magic_bonus: number | null        // 1, 2, or 3

// Charge capacity (static from item)
charges_max: number | null
recharge_formula: string | null   // "1d6+1"
recharge_timing: string | null    // "dawn"
```

### Top-level charges object (when backend Phase 3 complete)

```typescript
charges: {
  current: number | null
  max: number
  recharge_formula: string | null
  recharge_timing: string | null
} | null
```

## UI Layout

```
┌─────────────────────────────────────────────┐
│ +1 Longsword                            [X] │
├─────────────────────────────────────────────┤
│ [Melee Weapon] [Rare] [Attunement] [Main Hand] │  ← Existing badges
│ [+1 Magic]                                     │  ← NEW: Magic bonus badge
├─────────────────────────────────────────────┤
│ Weight: 3 lb        Value: 15 gp           │  ← Existing stats grid
│ Damage: 1d8+1 Slashing                     │  ← ENHANCED: Add damage type
│ Versatile: 1d10+1                          │  ← NEW: Versatile damage
│ Range: 150/600 ft                          │  ← NEW: For ranged weapons
├─────────────────────────────────────────────┤
│ ⚠️ Disadvantage on Stealth checks          │  ← Existing (armor)
│ ⚠️ Requires Strength 13                    │  ← Existing (armor)
│ DEX Bonus: +2 max                          │  ← NEW: For medium armor
├─────────────────────────────────────────────┤
│ Charges: [5] / 7  [−] [+]                  │  ← NEW: Interactive charges
│ Recharges 1d6+1 at dawn                    │  ← NEW: Recharge info
├─────────────────────────────────────────────┤
│ Description text here...                   │  ← Existing
├─────────────────────────────────────────────┤
│ Properties                                 │  ← Existing section
│ [Versatile] [Finesse] [Light]             │  ← Badges with tooltips
└─────────────────────────────────────────────┘
```

## Implementation Details

### Data Strategy

1. **Use equipment data first** - New fields come inline with equipment response
2. **Fetch `/items/{slug}` only for description** - Full item data has description text
3. **Graceful fallback** - Missing fields don't break display

### Charge Tracking

**When backend Phase 3 complete:**
- Display current/max charges with +/- buttons
- API: `PATCH /api/characters/{id}/counters/{counterId}` with `{ current: newValue }`
- Optimistic UI updates, revert on error

**Graceful fallback (until backend ready):**
- If `charges` object missing or `charges.current` is null
- Show static: "Max charges: 7 • Recharges 1d6+1 at dawn"
- No +/- buttons (read-only)

### Magic Bonus Badge

- Display "+1", "+2", "+3" badge for magic weapons/armor
- Use distinctive color (suggest `spell` or `primary`)
- Only show when `magic_bonus` is non-null

### DEX Bonus Display

| armor_type | max_dex_bonus | Display |
|------------|---------------|---------|
| light | null | (don't show) |
| medium | 2 | "DEX Bonus: +2 max" |
| heavy | 0 | "DEX Bonus: None" |

## Files to Modify

| File | Changes |
|------|---------|
| `app/components/character/inventory/ItemDetailModal.vue` | Add new field displays, charge UI |
| `app/types/character.ts` | Update CharacterEquipment type if needed |
| `tests/components/character/inventory/ItemDetailModal.test.ts` | Add tests for new fields |

## Test Strategy

1. **Weapon display** - Shows damage type, versatile damage from equipment data
2. **Ranged weapon** - Shows range in "normal/long ft" format
3. **Armor display** - Shows armor type, DEX cap, stealth disadvantage, STR requirement
4. **Magic item** - Shows magic bonus badge, rarity badge
5. **Charges static** - Shows max charges and recharge info when no current tracking
6. **Charges interactive** - +/- buttons work when charges.current available
7. **Graceful fallback** - Missing fields don't break the modal

## Related

- Backend plan: `docs/backend/plans/2025-12-18-issue-757-character-equipment-resource-enhancements.md`
- Backend handoff: See `../wrapper/.claude/handoffs.md`
- Issue: https://github.com/dfox288/ledger-of-heroes/issues/757
