# Backgrounds API Enhancement Proposals

**Date:** 2025-11-26
**Status:** Proposal
**API Endpoint:** `/api/v1/backgrounds`
**Overall Assessment:** 🟢 PASS - Excellent data structure with comprehensive character tables

---

## Executive Summary

The Backgrounds API is **production-ready** with exceptional handling of personality traits, ideals, bonds, flaws, and rollable tables. The random_tables system for character generation is particularly well-designed.

### Current Strengths
- Excellent random_tables for personality traits, ideals, bonds, flaws
- Complete equipment lists with item references
- Skill proficiencies with full skill objects
- Tool proficiency choices properly modeled
- Feature traits properly categorized
- Variant features supported (Noble's Retainers)
- Background-specific specialization tables (Criminal Specialty, Soldier Specialty)
- 35 backgrounds covering PHB and SCAG

### Minor Issues Found
- ~~Some backgrounds missing `languages` array entries (data in description only)~~ **FIXED**
- Tool proficiency choices could use better structure

### Recently Fixed ✅
- **Languages array now populated** - Sage and Acolyte now have `{ "is_choice": true, "quantity": 2 }` in languages array

---

## Logical Correctness Analysis ✅

### Skill Proficiency Verification (PHB Reference)

| Background | Skills | PHB Page | API Status |
|------------|--------|----------|------------|
| Acolyte | Insight, Religion | 127 | ✅ Correct |
| Criminal | Deception, Stealth | 129 | ✅ Correct |
| Noble | History, Persuasion | 135 | ✅ Correct |
| Sage | Arcana, History | 137 | ✅ Correct |
| Soldier | Athletics, Intimidation | 140 | ✅ Correct |
| Outlander | Athletics, Survival | 136 | ✅ Correct |

### Tool/Language Proficiency Verification - UPDATED 2025-11-26

| Background | Tools/Languages | PHB | API Status |
|------------|-----------------|-----|------------|
| Acolyte | 2 languages | 127 | ✅ **FIXED** - `is_choice: true, quantity: 2` |
| Criminal | Gaming set, Thieves' tools | 129 | ✅ Correct |
| Noble | Gaming set, 1 language | 135 | ✅ Correct |
| Sage | 2 languages | 137 | ✅ **FIXED** - `is_choice: true, quantity: 2` |
| Soldier | Gaming set, Vehicles (land) | 140 | ✅ Correct |
| Outlander | Musical instrument, 1 language | 136 | ✅ Correct |

### Starting Equipment Verification

| Background | Gold | PHB | API Status |
|------------|------|-----|------------|
| Acolyte | 15 gp | 127 | ✅ Correct |
| Criminal | 15 gp | 129 | ✅ Correct |
| Noble | 25 gp | 135 | ✅ (Check) |
| Sage | 10 gp | 137 | ✅ (Check) |

### Feature Verification

| Background | Feature | PHB | API Status |
|------------|---------|-----|------------|
| Acolyte | Shelter of the Faithful | 127 | ✅ Correct |
| Criminal | Criminal Contact | 129 | ✅ Correct |
| Noble | Position of Privilege | 135 | ✅ Correct |
| Sage | Researcher | 138 | ✅ (Check) |

---

## Structural Soundness Analysis ✅

### What Works Excellently

1. **Random Tables for Character Generation**
   ```json
   {
     "random_tables": [
       {
         "table_name": "Personality Trait",
         "dice_type": "d8",
         "entries": [
           { "roll_min": 1, "roll_max": 1, "result_text": "..." },
           { "roll_min": 2, "roll_max": 2, "result_text": "..." }
         ]
       },
       { "table_name": "Ideal", "dice_type": "d6", "entries": [...] },
       { "table_name": "Bond", "dice_type": "d6", "entries": [...] },
       { "table_name": "Flaw", "dice_type": "d6", "entries": [...] }
     ]
   }
   ```
   - Proper dice notation (d6, d8)
   - Roll ranges for weighted results
   - Supports background-specific tables (Criminal Specialty, Soldier Specialty)

2. **Skill Proficiency Structure**
   ```json
   {
     "proficiencies": [
       {
         "proficiency_type": "skill",
         "skill": {
           "id": 7,
           "name": "Insight",
           "slug": "insight",
           "ability_score": { "code": "WIS", "name": "Wisdom" }
         },
         "grants": true
       }
     ]
   }
   ```
   - Full skill object with ability score
   - `grants: true` indicates background provides this

3. **Equipment with Item References**
   ```json
   {
     "equipment": [
       {
         "item_id": 1907,
         "item": { "name": "Crowbar", "slug": "crowbar", ... },
         "quantity": 1
       },
       {
         "item_id": null,
         "description": "holy symbol (a gift to you when you entered the priesthood)",
         "quantity": 1
       }
     ]
   }
   ```
   - Links to actual item entities where possible
   - Fallback description for flavor items
   - Quantity support

4. **Trait Categorization**
   ```json
   {
     "traits": [
       { "name": "Description", "category": null },
       { "name": "Feature: Criminal Contact", "category": "feature" },
       { "name": "Suggested Characteristics", "category": "characteristics" },
       { "name": "Criminal Specialty", "category": "flavor" }
     ]
   }
   ```
   - `feature`: The mechanical background feature
   - `characteristics`: Personality tables
   - `flavor`: Optional customization (specialties)

5. **Language Choices**
   ```json
   {
     "languages": [
       { "is_choice": true }
     ]
   }
   ```
   - Supports language choice grants

---

## Feature Completeness Analysis ✅

### Essential Fields Present

| Field | Present | Notes |
|-------|---------|-------|
| name, slug | ✅ | All backgrounds |
| traits | ✅ | With categories |
| proficiencies | ✅ | Skills and tools |
| languages | ⚠️ | Some missing entries |
| equipment | ✅ | With item links |
| sources | ✅ | Book and page |

### Trait Categories Found

| Category | Purpose | Example |
|----------|---------|---------|
| `null` | Description/lore | "Description" |
| `feature` | Mechanical feature | "Feature: Shelter of the Faithful" |
| `characteristics` | Personality tables | "Suggested Characteristics" |
| `flavor` | Optional customization | "Criminal Specialty" |

---

## 🟡 Medium Priority Issues

### 1. Missing Language Entries

**Issue:** Some backgrounds have languages mentioned in description but empty `languages` array.

**Affected Backgrounds:**
- **Sage**: Should have `{ "is_choice": true }, { "is_choice": true }` (2 languages)
- **Acolyte**: Should have 2 language choices

**Current State (Sage):**
```json
{
  "languages": []
}
```
Description says: "Languages: Two of your choice"

**Expected:**
```json
{
  "languages": [
    { "is_choice": true },
    { "is_choice": true }
  ]
}
```

**Impact:** Character builders need this data to grant language proficiencies.

---

### 2. Tool Proficiency Choice Structure

**Issue:** Tool proficiency choices combine multiple items in one string.

**Current State (Soldier):**
```json
{
  "proficiency_type": "tool",
  "proficiency_subcategory": "gaming",
  "proficiency_name": "gaming set, vehicles (land)"
}
```

**Proposed Enhancement:**
```json
{
  "tool_proficiencies": [
    {
      "proficiency_type": "tool",
      "proficiency_subcategory": "gaming",
      "is_choice": true,
      "options": ["dice-set", "playing-card-set", "dragonchess-set"]
    },
    {
      "proficiency_type": "tool",
      "proficiency_subcategory": "vehicle",
      "proficiency_name": "Vehicles (land)",
      "is_choice": false
    }
  ]
}
```

**Impact:** Clearer separation allows character builders to present proper choices.

---

## Medium Priority Enhancements

### 3. Add `feature_name` Top-Level Field

**Current State:** Feature buried in traits array.

**Proposed Enhancement:**
```json
{
  "feature_name": "Shelter of the Faithful",
  "feature_description": "As an acolyte, you command the respect..."
}
```

**Benefit:** Quick access to background feature without parsing traits.

---

### 4. Add `starting_gold` Computed Field

**Current State:** Gold is an equipment item with quantity.

**Proposed Enhancement:**
```json
{
  "starting_gold": 15
}
```

**Benefit:** Easier character sheet generation.

---

### 5. Add `skill_count` and `skill_names` Summary Fields

**Proposed Enhancement:**
```json
{
  "skill_names": ["Insight", "Religion"],
  "skill_count": 2
}
```

**Benefit:** Quick summary for list views without parsing proficiencies.

---

## Low Priority / Nice-to-Have

### 6. Extract Ideals Alignment Tags

**Current State:** Alignment in parentheses: `"Tradition. The ancient traditions... (Lawful)"`

**Proposed Enhancement:**
```json
{
  "entries": [
    {
      "roll_min": 1,
      "result_text": "Tradition. The ancient traditions...",
      "alignment": "Lawful"
    }
  ]
}
```

**D&D Context:** Ideals map to alignments, useful for character generation.

---

### 7. Add Variant Background Flag

**Current State:** Variants like "Spy" (Criminal variant) are separate entries or in description.

**Proposed Enhancement:**
```json
{
  "is_variant": true,
  "parent_background_slug": "criminal"
}
```

**Benefit:** Shows relationship between base backgrounds and variants.

---

### 8. Add `source_type` Field

**Proposed Enhancement:**
```json
{
  "source_type": "core",  // core, supplement, adventure
  "source_book": "PHB"
}
```

| Source Type | Examples |
|-------------|----------|
| `core` | PHB backgrounds |
| `supplement` | SCAG, XGE backgrounds |
| `adventure` | Module-specific backgrounds |

**Benefit:** Filter by source type for campaign restrictions.

---

## Implementation Priority Matrix

| Enhancement | Effort | Impact | Priority |
|-------------|--------|--------|----------|
| ~~Fix missing language entries~~ | ~~Low~~ | ~~High~~ | ✅ **DONE** |
| Improve tool proficiency structure | Medium | Medium | 🟡 Medium |
| Add `feature_name` field | Low | Medium | 🟡 Medium |
| Add `starting_gold` field | Low | Medium | 🟡 Medium |
| Add skill summary fields | Low | Low | 🟢 Low |
| Extract alignment from ideals | Medium | Low | 🟢 Low |
| Add variant background flag | Low | Low | 🟢 Low |
| Add `source_type` field | Low | Low | 🟢 Low |

---

## Verification Checklist

The following was verified against official D&D 5e sources:

**Skill Proficiencies:**
- [x] Acolyte: Insight, Religion (PHB p.127)
- [x] Criminal: Deception, Stealth (PHB p.129)
- [x] Noble: History, Persuasion (PHB p.135)
- [x] Sage: Arcana, History (PHB p.137)
- [x] Soldier: Athletics, Intimidation (PHB p.140)

**Tool Proficiencies:**
- [x] Criminal: Gaming set (choice), Thieves' tools (PHB p.129)
- [x] Noble: Gaming set (choice) (PHB p.135)
- [x] Soldier: Gaming set (choice), Vehicles (land) (PHB p.140)

**Features:**
- [x] Acolyte: Shelter of the Faithful (PHB p.127)
- [x] Criminal: Criminal Contact (PHB p.129)
- [x] Noble: Position of Privilege (PHB p.135)

**Random Tables:**
- [x] All PHB backgrounds have d8 Personality Traits
- [x] All PHB backgrounds have d6 Ideals, Bonds, Flaws
- [x] Criminal Specialty table (d8) present
- [x] Soldier Specialty table (d8) present

---

## Summary

The Backgrounds API is **production-ready** with excellent modeling of:

- Character personality tables (traits, ideals, bonds, flaws)
- Rollable tables for character generation
- Skill and tool proficiencies
- Starting equipment with item references
- Background features and variants

The random_tables system is particularly well-designed, supporting both standard d6/d8 personality tables and background-specific customization tables (Criminal Specialty, Soldier Specialty).

The main data gap is **missing language proficiency entries** for some backgrounds - the data exists in description text but not in the structured `languages` array. This should be a high-priority fix for character builder functionality.

---

## Related Documentation

- Frontend background pages: `app/pages/backgrounds/`
- Background filter store: `app/stores/backgroundFilters.ts`
- Backend API: `/Users/dfox/Development/dnd/importer`
