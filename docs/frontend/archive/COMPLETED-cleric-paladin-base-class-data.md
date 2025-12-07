# RESOLVED: Cleric and Paladin Base Class Data Missing

**Created:** 2025-11-26
**Resolved:** 2025-11-26
**Status:** RESOLVED - Backend fix applied
**Priority:** HIGH
**Affects:** Frontend class detail pages for Cleric and Paladin

---

## Resolution

**Root Cause:** Import file ordering issue. DMG files (containing only subclass data like Death Domain and Oathbreaker) were imported alphabetically before PHB files (containing full base class data). The merge logic only added subclasses, not base class relationships.

**Fix:** Updated `ClassImporter::mergeSupplementData()` to also import base class relationships (traits, proficiencies, features, spell progression, counters, equipment, description) when the existing class is missing them.

**Verified Results:**
| Class | Features | Proficiencies | Traits | Level Progression | Counters |
|-------|----------|---------------|--------|-------------------|----------|
| Cleric | 25 ✅ | 11 ✅ | 4 ✅ | 20 ✅ | 3 ✅ |
| Paladin | 30 ✅ | 14 ✅ | 4 ✅ | 19 ✅ | 22 ✅ |

**Action Required:** Re-fetch API data for Cleric and Paladin on frontend.

---

## Summary

The Cleric and Paladin base class records in the API are missing essential data including features, proficiencies, level progression (spell slots), traits, and descriptions. The subclasses (domains/oaths) are fully populated, but the parent class records are stub entries.

---

## Evidence

### API Response Analysis

**Cleric (`/api/v1/classes/cleric`):**
```json
{
  "name": "Cleric",
  "description": "No description available",
  "hit_die": 8,                    // ✅ Correct
  "spellcasting_ability": "WIS",   // ✅ Correct
  "computed": {
    "section_counts": {
      "features": 0,        // ❌ Should be ~15
      "proficiencies": 0,   // ❌ Should be ~12
      "traits": 0,          // ❌ Should have class lore
      "subclasses": 14,     // ✅ All domains present
      "spells": 116,        // ✅ Spell list present
      "counters": 0         // ❌ Missing Channel Divinity
    }
  }
}
```

**Paladin (`/api/v1/classes/paladin`):**
```json
{
  "name": "Paladin",
  "description": "No description available",
  "hit_die": 10,                   // ✅ Correct
  "spellcasting_ability": "CHA",   // ✅ Correct
  "proficiencies": 0,              // ❌ Should be ~15
  "features": 0,                   // ❌ Should be ~20
  "level_progression": 0           // ❌ Should be 20 rows (half-caster slots)
}
```

### Comparison with Working Classes

| Class | Description | Proficiencies | Features | Level Progression |
|-------|-------------|---------------|----------|-------------------|
| Wizard | ✅ 1499 chars | ✅ 13 | ✅ 16 | ✅ 20 |
| Fighter | ✅ 1386 chars | ✅ 16 | ✅ 31 | ✅ 20 |
| Barbarian | ✅ 1066 chars | ✅ 13 | ✅ 26 | ✅ 20 |
| **Cleric** | ❌ 24 chars | ❌ 0 | ❌ 0 | ❌ 0 |
| **Paladin** | ❌ 24 chars | ❌ 0 | ❌ 0 | ❌ 0 |

---

## What Should Be Present

### Cleric Base Class (PHB p. 56-58)

**Proficiencies:**
- Armor: Light armor, medium armor, shields
- Weapons: Simple weapons
- Saving Throws: Wisdom, Charisma
- Skills: Choose 2 from History, Insight, Medicine, Persuasion, Religion

**Features:**
| Level | Feature |
|-------|---------|
| 1 | Spellcasting, Divine Domain |
| 2 | Channel Divinity (1/rest), Divine Domain feature |
| 4 | Ability Score Improvement |
| 5 | Destroy Undead (CR 1/2) |
| 6 | Channel Divinity (2/rest), Divine Domain feature |
| 8 | Ability Score Improvement, Destroy Undead (CR 1), Divine Domain feature |
| 10 | Divine Intervention |
| 11 | Destroy Undead (CR 2) |
| 12 | Ability Score Improvement |
| 14 | Destroy Undead (CR 3) |
| 16 | Ability Score Improvement |
| 17 | Destroy Undead (CR 4), Divine Domain feature |
| 18 | Channel Divinity (3/rest) |
| 19 | Ability Score Improvement |
| 20 | Divine Intervention improvement |

**Level Progression:** Full caster spell slot table (same as Wizard)

**Counters:**
- Channel Divinity: 1 at level 2, 2 at level 6, 3 at level 18 (resets on Short Rest)
- Divine Intervention: 1 per Long Rest (at level 20, automatic success)

### Paladin Base Class (PHB p. 82-85)

**Proficiencies:**
- Armor: All armor, shields
- Weapons: Simple weapons, martial weapons
- Saving Throws: Wisdom, Charisma
- Skills: Choose 2 from Athletics, Insight, Intimidation, Medicine, Persuasion, Religion

**Features:**
| Level | Feature |
|-------|---------|
| 1 | Divine Sense, Lay on Hands |
| 2 | Fighting Style, Spellcasting, Divine Smite |
| 3 | Divine Health, Sacred Oath |
| 4 | Ability Score Improvement |
| 5 | Extra Attack |
| 6 | Aura of Protection |
| 7 | Sacred Oath feature |
| 8 | Ability Score Improvement |
| 9 | — |
| 10 | Aura of Courage |
| 11 | Improved Divine Smite |
| 12 | Ability Score Improvement |
| 14 | Cleansing Touch |
| 15 | Sacred Oath feature |
| 16 | Ability Score Improvement |
| 18 | Aura improvements (30 ft) |
| 19 | Ability Score Improvement |
| 20 | Sacred Oath feature |

**Level Progression:** Half-caster spell slot table (starts at level 2)

**Counters:**
- Divine Sense: CHA modifier + 1 per Long Rest
- Lay on Hands: 5 × Paladin level HP pool per Long Rest
- Channel Divinity: 1 per Short Rest (from level 3)
- Cleansing Touch: CHA modifier per Long Rest

---

## Secondary Issues Found

### 1. Subclass `hit_die: 0`

Several subclasses have `hit_die: 0` instead of inheriting from parent:
- `cleric-death-domain`: `hit_die: 0` (should be 8)
- `cleric-arcana-domain`: `hit_die: 0` (should be 8)
- `paladin-oathbreaker`: `hit_die: 0` (should be 10)

The `computed.hit_points` is also `null` for these subclasses.

### 2. Progression Tables Show "—" for All Features

The subclass progression tables show `features: "—"` for every level, even levels where the subclass grants features. This may be intentional (features listed separately) but could confuse users.

---

## Impact on Frontend

1. **Class detail pages** for Cleric and Paladin will display:
   - No proficiencies section
   - No features/class abilities section
   - No spell slot progression table
   - "No description available" instead of PHB flavor text

2. **Character builders** would be unable to show:
   - What armor/weapons a Cleric can use
   - When Channel Divinity unlocks
   - Spell slot progression for prepared casters

3. **Subclass selection** works fine since domain/oath data is present

---

## Recommended Fix

In the Laravel importer (`../importer`):

1. Ensure the base class import includes:
   - `proficiencies` relationship data
   - `features` relationship data
   - `level_progression` entries (spell slots)
   - `traits` for class lore/description
   - `counters` for class resources
   - Full `description` text

2. Verify the import source has Cleric/Paladin base class data (not just subclass data)

3. Fix `hit_die: 0` on subclasses - either:
   - Inherit from parent via `inherited_data`
   - Or populate with correct value

---

## Workaround (Frontend)

Until fixed, could potentially:
1. Display a "Data incomplete" notice on affected pages
2. Link to external SRD for missing info
3. Use `inherited_data` pattern to pull from first subclass (not ideal)

---

## Related Files

- Backend: `../importer/` (Laravel importer)
- Frontend affected: `app/pages/classes/[slug].vue`
- API endpoint: `GET /api/v1/classes/{slug}`
