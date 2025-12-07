# API Verification Report: Classes Endpoint

**Date:** 2025-11-26
**Endpoint:** `/api/v1/classes`
**Focus:** Paladin/Cleric Fix Verification + Full Class Analysis
**Status:** PASS (with minor remaining issues)

---

## Summary

The backend fix for missing Cleric and Paladin base class data has been **successfully applied**.

**Root Cause:** Import file ordering issue where DMG files (containing only subclass data like Death Domain and Oathbreaker) were imported alphabetically before PHB files (containing full base class data). The merge logic only added subclasses, not base class relationships.

**Fix Applied:** Updated `ClassImporter::mergeSupplementData()` to also import base class relationships (traits, proficiencies, features, spell progression, counters, equipment, description) when the existing class is missing them.

---

## Verified Results (Cleric & Paladin)

| Class | Features | Proficiencies | Traits | Level Progression | Counters | Description |
|-------|----------|---------------|--------|-------------------|----------|-------------|
| **Cleric** | 25 | 11 | 4 | 20 | 3 | 765 chars |
| **Paladin** | 30 | 14 | 4 | 19 | 22 | 1264 chars |

Both classes now have complete data matching their PHB entries.

---

## Logical Correctness Analysis

### Paladin (PHB p. 82-85)

| Check | Status | Notes |
|-------|--------|-------|
| Hit die | d10 | Correct |
| Spellcasting ability | CHA | Correct |
| Spell slot progression | Half-caster | Correct - starts at level 2, max 5th at level 17 |
| Armor proficiencies | All + shields | Correct |
| Weapon proficiencies | Simple + martial | Correct |
| Saving throws | WIS, CHA | Correct |
| Skill choices | 6 options, pick 2 | Correct (Athletics, Insight, Intimidation, Medicine, Persuasion, Religion) |

### Cleric (PHB p. 56-58)

| Check | Status | Notes |
|-------|--------|-------|
| Hit die | d8 | Correct |
| Spellcasting ability | WIS | Correct |
| Spell slot progression | Full caster | Correct - 9th-level slots at level 17 |
| Armor proficiencies | Light, medium, shields | Correct (no heavy armor) |
| Weapon proficiencies | Simple only | Correct (no martial) |
| Saving throws | WIS, CHA | Correct |
| Skill choices | 5 options, pick 2 | Correct (History, Insight, Medicine, Persuasion, Religion) |

---

## All Base Classes Status

| Class | Features | Proficiencies | Level Prog | Counters | Traits | Status |
|-------|----------|---------------|------------|----------|--------|--------|
| Artificer | 24 | 16 | 20 | 7 | 3 | Complete |
| Barbarian | 26 | 13 | 0 | 5 | 4 | No progression table |
| Bard | 29 | 27 | 20 | 0 | 4 | Complete |
| **Cleric** | 25 | 11 | 20 | 3 | 4 | **FIXED** |
| Druid | 20 | 24 | 20 | 2 | 4 | Complete |
| Fighter | 31 | 16 | 0 | 6 | 4 | No progression table |
| Monk | 37 | 11 | 0 | 20 | 4 | No progression table |
| **Paladin** | 30 | 14 | 19 | 22 | 4 | **FIXED** |
| Ranger | 31 | 15 | 19 | 0 | 4 | Complete |
| Rogue | 34 | 20 | 0 | 1 | 4 | No progression table |
| Sorcerer | 42 | 13 | 20 | 19 | 4 | Complete |
| Warlock | 29 | 11 | 20 | 8 | 4 | Complete |
| Wizard | 16 | 13 | 20 | 1 | 4 | Complete |

---

## Remaining Issues

### 1. Subclass `hit_die: 0` (23 subclasses affected)

**Priority:** LOW

**What's Wrong:** Many subclasses have `hit_die: 0` in their direct field, though the correct value is available in `inherited_data.hit_die`.

**Examples:**
- `cleric-death-domain`: `hit_die: 0` (inherited: 8)
- `cleric-arcana-domain`: `hit_die: 0` (inherited: 8)
- `paladin-oathbreaker`: `hit_die: 0` (inherited: 10)

**D&D Context:** Subclasses don't change hit die - they inherit from their parent class. A Death Domain Cleric still uses a d8, an Oathbreaker Paladin still uses a d10.

**API Impact:** Frontend can work around this using `inherited_data.hit_die`, but the direct field is confusing.

**Suggested Fix:** Either:
1. Populate `hit_die` with parent's value during import
2. Or make `hit_die` nullable for subclasses (indicate it's inherited)

---

### 2. Non-Caster Classes Missing `level_progression`

**Priority:** MEDIUM

**Affected Classes:** Barbarian, Fighter, Monk, Rogue (+ Sidekicks)

**What's Wrong:** These classes have empty `level_progression` arrays because they don't have spell slots.

**D&D Context:** Non-spellcasting classes have class-specific progressions that should be shown:
- **Barbarian:** Rages per day, Rage damage bonus
- **Fighter:** Action Surge uses, Indomitable uses
- **Monk:** Ki points, Martial Arts die, Unarmored Movement bonus
- **Rogue:** Sneak Attack dice

**Current State:** This data appears to be tracked via `counters` instead (e.g., Monk has 20 counter entries).

**Suggested Fix:** Consider generating a computed `progression_table` for non-casters that shows their class resources per level, similar to how casters show spell slots.

---

## Structural Soundness

| Aspect | Status | Notes |
|--------|--------|-------|
| Consistent data structure | Pass | All base classes use same schema |
| Normalization | Pass | Proficiency types properly linked |
| Computed fields | Pass | `hit_points`, `progression_table` computed correctly |
| Relationships | Pass | Subclasses properly nested under parent |
| Inherited data | Pass | Subclasses have `inherited_data` block |

---

## Feature Completeness

### Base Classes Now Have:
- Full descriptions (no more "No description available")
- Proficiencies (armor, weapons, saving throws, skill choices)
- Features (class abilities by level)
- Level progression (spell slots for casters)
- Counters (class resource tracking)
- Traits (class lore)
- Computed hit points with proper formulas

---

## Frontend Impact

**No code changes required.** The frontend uses dynamic API fetching, so the corrected data is automatically available on page refresh. The fix is live and working.

---

## Related Documents

- Original issue: `docs/proposals/COMPLETED-cleric-paladin-base-class-data.md`
- Backend fix location: `../importer/app/Importers/ClassImporter.php`
