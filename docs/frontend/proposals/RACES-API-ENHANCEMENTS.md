# Races API Enhancement Proposals

**Date:** 2025-11-26
**Status:** Proposal
**API Endpoint:** `/api/v1/races`
**Overall Assessment:** 🟢 PASS - Excellent data structure with comprehensive racial trait modeling

---

## Executive Summary

The Races API is **production-ready** with comprehensive modeling of D&D 5e racial features including ability score modifiers, innate spellcasting, proficiencies, damage resistances, condition immunities, and subrace hierarchies.

### Current Strengths
- Excellent ability score modifier system with choice support
- Comprehensive innate spellcasting with level requirements and usage limits
- Proper subrace/parent race relationships
- Damage resistance and condition immunity modeling
- Language support with choice options
- Proficiency system (skills, weapons, tools)
- Trait categorization (species, subspecies, description)

### Minor Issues Found
- Base races (Elf, Dwarf) have empty traits/modifiers - data exists on subraces
- Some naming inconsistency (subrace names missing parent prefix)

---

## Logical Correctness Analysis ✅

### Ability Score Verification (PHB Reference)

| Race | Modifiers | PHB Page | API Status |
|------|-----------|----------|------------|
| High Elf | DEX +2, INT +1 | 23 | ✅ Correct |
| Hill Dwarf | CON +2, WIS +1 | 20 | ✅ Correct |
| Human | All +1 | 29 | ✅ Correct |
| Dragonborn | STR +2, CHA +1 | 32 | ✅ Correct |
| Tiefling | CHA +2, INT +1 | 42 | ✅ Correct |
| Half-Elf | CHA +2, Any 2 +1 | 38 | ✅ Correct (with choice) |

### Speed Verification

| Race | Speed | PHB Page | API Status |
|------|-------|----------|------------|
| Elf | 30 ft | 23 | ✅ Correct |
| Dwarf | 25 ft | 20 | ✅ Correct |
| Wood Elf | 35 ft | 24 | ✅ Correct |
| Human | 30 ft | 29 | ✅ Correct |
| Halfling | 25 ft | 26 | ✅ Correct |

### Size Verification

| Race | Size | PHB Page | API Status |
|------|------|----------|------------|
| Elf | Medium | 23 | ✅ Correct |
| Dwarf | Medium | 20 | ✅ Correct |
| Halfling | Small | 26 | ✅ Correct |
| Gnome | Small | 35 | ✅ Correct |

### Innate Spellcasting Verification

| Race | Spells | Level Req | API Status |
|------|--------|-----------|------------|
| Tiefling | Thaumaturgy (cantrip) | - | ✅ Correct |
| Tiefling | Hellish Rebuke (1/LR) | 3rd | ✅ Correct |
| Tiefling | Darkness (1/LR) | 5th | ✅ Correct |
| Aasimar (DMG) | Light (cantrip) | - | ✅ Correct |
| Aasimar (DMG) | Lesser Restoration (1/LR) | 3rd | ✅ Correct |
| Aasimar (DMG) | Daylight (1/LR) | 5th | ✅ Correct |

### Damage Resistance Verification

| Race | Resistance | PHB Page | API Status |
|------|------------|----------|------------|
| Tiefling | Fire | 43 | ✅ Correct |
| Hill Dwarf | Poison | 20 | ✅ Correct |

### Proficiency Verification

| Race | Proficiencies | PHB Page | API Status |
|------|---------------|----------|------------|
| High Elf | Perception, Longsword, Shortsword, Shortbow, Longbow | 23-24 | ✅ Correct |
| Hill Dwarf | Battleaxe, Handaxe, Light Hammer, Warhammer | 20 | ✅ Correct |

---

## Structural Soundness Analysis ✅

### What Works Excellently

1. **Ability Score Modifier System**
   ```json
   {
     "modifiers": [
       {
         "modifier_category": "ability_score",
         "ability_score": { "code": "DEX", "name": "Dexterity" },
         "value": "2"
       },
       {
         "modifier_category": "ability_score",
         "value": "+1",
         "is_choice": true,
         "choice_count": 2,
         "choice_constraint": "different"
       }
     ]
   }
   ```
   - Handles fixed bonuses (+2 DEX)
   - Supports choice-based bonuses (Half-Elf: any 2 different +1)
   - Properly links to ability score reference data

2. **Innate Spellcasting System**
   ```json
   {
     "spells": [
       {
         "spell": { "name": "Hellish Rebuke", "level": 1 },
         "ability_score": { "code": "CHA" },
         "level_requirement": 3,
         "usage_limit": "1/long rest",
         "is_cantrip": false
       }
     ]
   }
   ```
   - Full spell object embedded
   - Spellcasting ability specified
   - Level requirements for racial spells
   - Usage limits (1/long rest, at will)
   - Cantrip flag

3. **Subrace Hierarchy**
   ```json
   {
     "parent_race": {
       "id": 12,
       "slug": "elf",
       "name": "Elf"
     },
     "subraces": []
   }
   ```
   - Base races link to subraces
   - Subraces link back to parent
   - Traits properly combined (species + subspecies categories)

4. **Damage Resistance/Immunity Modeling**
   ```json
   {
     "modifiers": [
       {
         "modifier_category": "damage_resistance",
         "damage_type": { "code": "F", "name": "Fire" },
         "value": "resistance"
       }
     ]
   }
   ```

5. **Condition Effect Modeling**
   ```json
   {
     "conditions": [
       {
         "condition": { "name": "Charmed", "description": "..." },
         "effect_type": "advantage"
       }
     ]
   }
   ```
   - Fey Ancestry (advantage vs charmed)
   - Dwarven Resilience (advantage vs poison)

6. **Language System**
   ```json
   {
     "languages": [
       { "language": { "name": "Common" }, "is_choice": false },
       { "language": { "name": "Elvish" }, "is_choice": false },
       { "is_choice": true }  // Extra language choice
     ]
   }
   ```

7. **Trait Categorization**
   ```json
   {
     "traits": [
       { "name": "Darkvision", "category": "species" },
       { "name": "Elf Weapon Training", "category": "subspecies" },
       { "name": "Description", "category": "description" }
     ]
   }
   ```
   - `species`: Base race traits
   - `subspecies`: Subrace-specific traits
   - `description`: Flavor/lore text

---

## Feature Completeness Analysis ✅

### Essential Fields Present

| Field | Present | Notes |
|-------|---------|-------|
| name, slug | ✅ | All races |
| size | ✅ | Nested object with code/name (S, M, L) |
| speed | ✅ | Integer in feet |
| traits | ✅ | Array with categories |
| modifiers | ✅ | Ability scores, damage resist, etc. |
| proficiencies | ✅ | Skills, weapons, tools |
| languages | ✅ | With choice support |
| spells | ✅ | Innate spellcasting |
| conditions | ✅ | Advantage/immunity to conditions |
| parent_race | ✅ | For subraces |
| subraces | ✅ | For base races |
| sources | ✅ | Book and page references |

---

## Medium Priority Enhancements

### 1. Populate Base Race Data

**Current State:**
Base races (Elf, Dwarf, etc.) return empty `traits` and `modifiers`. All data is on subraces.

**Example (Elf base):**
```json
{
  "name": "Elf",
  "traits": [],
  "modifiers": []
}
```

**Proposed Enhancement:**
Option A: Populate shared traits on base race, subrace-specific traits on subraces
Option B: Add `computed.inherited_traits` on subraces that includes parent traits

**D&D Context:**
PHB presents base race traits separately from subrace traits. Darkvision, Fey Ancestry, Trance are all Elf traits, not High Elf traits.

**Benefit:** Matches PHB structure, reduces data duplication, clarifies what comes from where.

---

### 2. ~~Add `is_subrace` Boolean Flag~~ ✅ IMPLEMENTED

**Status:** ✅ Implemented 2025-11-26

**Current State (NOW WORKING):**
```json
{
  "is_subrace": false  // for base races like Aarakocra
}
// or
{
  "is_subrace": true   // for subraces like Aarakocra (DMG)
}
```

**Verified Examples:**
| Race | is_subrace | parent_race |
|------|------------|-------------|
| Aarakocra | `false` | null |
| Aarakocra (DMG) | `true` | Aarakocra |
| Aasimar | `false` | null |

**Benefit:** Simplifies frontend filtering and display logic. ✅

---

### 3. Add `darkvision_range` Field

**Current State:** Darkvision range only in trait description text.

**Proposed Enhancement:**
```json
{
  "darkvision_range": 60,  // or 120 for Drow
  "superior_darkvision": false
}
```

**D&D Context:**
- Most races: 60 ft darkvision
- Drow: 120 ft (Superior Darkvision)
- Some subterranean races: 120 ft

**Benefit:** Enables filtering by darkvision capability, important for dungeon-focused campaigns.

---

### 4. Add `flight_speed` and `swim_speed` Fields

**Current State:** Only `speed` (walking) available. Flight/swim in traits.

**Proposed Enhancement:**
```json
{
  "speed": 30,
  "fly_speed": 50,
  "swim_speed": null,
  "speed_notes": "To use flying speed, can't wear medium or heavy armor"
}
```

**Relevant Races:**
- Aarakocra: 50 ft fly (no medium/heavy armor)
- Triton: 30 ft swim
- Sea Elf: 30 ft swim

**Benefit:** Enables "Show races with flight" filter.

---

### 5. Standardize Subrace Names

**Current State:** Inconsistent naming (some include parent, some don't).

**Examples:**
```
"elf-high" → "High" (missing "Elf")
"halfling-lightfoot" → "Lightfoot" (missing "Halfling")
```

**Proposed Enhancement:**
```json
{
  "name": "High Elf",
  "short_name": "High",
  "display_name": "Elf (High)"
}
```

**Benefit:** Consistent display in UI, clear parent/subrace relationship.

---

## Low Priority / Nice-to-Have

### 6. Add `age_maturity` and `age_lifespan` Fields

**Current State:** Age info only in description trait.

**Proposed Enhancement:**
```json
{
  "age_maturity": 100,
  "age_lifespan": 750,
  "age_description": "Elves can live to be 750 years old"
}
```

**Benefit:** Enables age-related character generation features.

---

### 7. Add `creature_type` Field

**Current State:** All races assumed to be humanoid.

**Proposed Enhancement:**
```json
{
  "creature_type": "humanoid",  // or "fey" for some newer races
  "creature_subtypes": ["elf"]
}
```

**D&D Context:**
- Most races: Humanoid
- Newer races (Fairy, Satyr): Fey
- Affects spell targeting (Hold Person vs Hold Monster)

---

### 8. Add `tool_proficiency_choices` Structure

**Current State:** Dwarf tool choice described in trait text.

**Proposed Enhancement:**
```json
{
  "tool_proficiency_choice": {
    "count": 1,
    "options": ["smith's-tools", "brewer's-supplies", "mason's-tools"]
  }
}
```

**Benefit:** Character builder can present tool choices.

---

## Implementation Priority Matrix

| Enhancement | Effort | Impact | Priority |
|-------------|--------|--------|----------|
| Populate base race data | Medium | High | 🟡 Medium |
| ~~Add `is_subrace` flag~~ | ~~Low~~ | ~~Medium~~ | ✅ **DONE** |
| Add `darkvision_range` | Low | Medium | 🟡 Medium |
| Add `fly_speed`/`swim_speed` | Low | Medium | 🟡 Medium |
| Standardize subrace names | Medium | Medium | 🟡 Medium |
| Add age fields | Low | Low | 🟢 Low |
| Add `creature_type` | Low | Low | 🟢 Low |
| Add tool proficiency choices | Low | Low | 🟢 Low |

---

## Verification Checklist

The following was verified against official D&D 5e sources:

**Ability Scores:**
- [x] High Elf: DEX +2, INT +1 (PHB p.23)
- [x] Hill Dwarf: CON +2, WIS +1 (PHB p.20)
- [x] Human: All +1 (PHB p.29)
- [x] Dragonborn: STR +2, CHA +1 (PHB p.32)
- [x] Tiefling: CHA +2, INT +1 (PHB p.42)
- [x] Half-Elf: CHA +2, choice of 2 others +1 (PHB p.38)

**Speed:**
- [x] Elf/Human: 30 ft
- [x] Dwarf/Halfling: 25 ft
- [x] Wood Elf: 35 ft

**Size:**
- [x] Halfling/Gnome: Small
- [x] All others checked: Medium

**Innate Spellcasting:**
- [x] Tiefling: Thaumaturgy (cantrip), Hellish Rebuke (3rd, 1/LR), Darkness (5th, 1/LR)
- [x] Aasimar (DMG): Light, Lesser Restoration, Daylight

**Proficiencies:**
- [x] High Elf: Perception, Longsword, Shortsword, Shortbow, Longbow
- [x] Hill Dwarf: Battleaxe, Handaxe, Light Hammer, Warhammer

---

## Summary

The Races API is **production-ready** with comprehensive D&D 5e racial data. The modifier system elegantly handles:

- Fixed ability score bonuses
- Choice-based bonuses (Half-Elf)
- Damage resistances and immunities
- Condition advantages

The innate spellcasting system properly models:
- Level requirements
- Usage limits (at will, 1/long rest)
- Spellcasting ability

The suggested enhancements are quality-of-life improvements for character builders, not corrections to data accuracy. The most impactful would be populating base race data and adding special movement speeds (flight, swim).

---

## Related Documentation

- Frontend race pages: `app/pages/races/`
- Race filter store: `app/stores/raceFilters.ts`
- Backend API: `/Users/dfox/Development/dnd/importer`
