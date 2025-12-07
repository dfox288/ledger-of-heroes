# Complete Filter Action Plans - All Entities

**Date:** 2025-11-25
**Purpose:** Detailed action plans for each entity showing what needs to be fixed, added, or removed

---

## 1. ITEMS Page Action Plan

### Current State
- **Filters Implemented:** 12
- **Status:** Working, but missing weapon/armor shopping filters

### Actions Required

#### ADD (6 HIGH priority filters):
1. **strength_requirement** (141 items)
   - Type: Toggle (Any / STR 13+ / STR 15+)
   - Location: QUICK section
   - Field: `strength_requirement`
   - Use: Armor shopping for low-STR characters

2. **damage_dice** (300+ items)
   - Type: Multiselect (1d4, 1d6, 1d8, 1d10, 1d12, 2d6)
   - Location: ADVANCED section
   - Field: `damage_dice`
   - Use: Weapon shopping by damage

3. **versatile_damage** (81 items)
   - Type: Multiselect (1d8, 1d10, 1d12)
   - Location: ADVANCED section
   - Field: `versatile_damage`
   - Use: Versatile weapon optimization

4. **range_normal** + **range_long** (226 items)
   - Type: Single select with ranges
   - Options: Any / Short <30ft / Medium 30-80ft / Long 80-150ft / Very Long >150ft
   - Location: ADVANCED section
   - Fields: `range_normal` AND `range_long`
   - Use: Ranged weapon shopping

5. **recharge_timing** (189+ items)
   - Type: Multiselect
   - Options: Dawn (discovered), TODO: find other values
   - Location: ADVANCED section
   - Field: `recharge_timing`
   - Use: Magic item resource planning

#### REMOVE (1 broken filter):
- **has_prerequisites** toggle
  - Reason: Backend doesn't support `prerequisites` field filtering
  - Action: Remove UI component, state, and chip
  - Alternative: Document in tooltip "Prerequisites shown in detail view"

#### FIX: None needed

---

## 2. MONSTERS Page Action Plan

### Current State
- **Filters Implemented:** 12 (all working)
- **Critical Bugs:** 5 filters missing chips
- **Status:** Functional but incomplete UX

### Actions Required

#### FIX (5 CRITICAL chip bugs):
1. **Alignment** - Add missing chips for selected alignments
2. **Has Fly** - Add chip when filter active
3. **Has Swim** - Add chip when filter active
4. **Has Burrow** - Add chip when filter active
5. **Has Climb** - Add chip when filter active

#### ADD (6 HIGH priority filters):
1. **armor_type** (289 items - "natural armor")
   - Type: Multiselect
   - Options: Natural Armor, Plate, etc. (TODO: discover all values)
   - Field: `armor_type`

2. **can_hover** (3 items)
   - Type: Boolean toggle
   - Field: `can_hover`

3. **has_lair_actions** (45 items)
   - Type: Boolean toggle
   - Field: `has_lair_actions`
   - Note: Returns null in response but filter WORKS

4. **has_reactions** (34 items)
   - Type: Boolean toggle
   - Field: `has_reactions`

5. **is_spellcaster** (129 items)
   - Type: Boolean toggle
   - Field: `is_spellcaster`

6. **has_magic_resistance** (85 items)
   - Type: Boolean toggle
   - Field: `has_magic_resistance`

#### REMOVE: None

---

## 3. RACES Page Action Plan

### Current State
- **Filters Implemented:** 9
- **Status:** Good, but missing family browsing

### Actions Required

#### ADD (1 HIGH priority filter):
1. **parent_race_name**
   - Type: Dropdown
   - Options: Fetch base races, use names as values
   - Field: `parent_race_name`
   - Use: "Show all Elf variants" → High Elf, Wood Elf, Drow, Eladrin
   - Implementation: Use `useReferenceData` to fetch base races

#### FIX: None

#### REMOVE: None

---

## 4. CLASSES Page Action Plan

### Current State
- **Filters Implemented:** 6
- **Status:** Good foundation, missing proficiency filters

### Actions Required

#### ADD (5 filters - 3 HIGH, 2 MEDIUM priority):

**HIGH Priority (3):**
1. **max_spell_level**
   - Type: Dropdown
   - Options: All / 9th (Full Caster) / 5th (Half Caster) / 4th (Third Caster) / 0 (Non-Caster)
   - Field: `max_spell_level`
   - Use: Multiclass planning, spellcasting power

2. **armor_proficiencies**
   - Type: Multiselect
   - Options: Light Armor, Medium Armor, Heavy Armor, Shields
   - Field: `armor_proficiencies` (IN array)
   - Use: AC optimization, multiclass planning

3. **weapon_proficiencies** 
   - Type: Multiselect
   - Options: Simple Weapons, Martial Weapons
   - Field: `weapon_proficiencies` (IN array)
   - Use: Weapon access, multiclass planning

**MEDIUM Priority (2):**
4. **skill_proficiencies**
   - Type: Multiselect
   - Options: All 18 skills
   - Field: `skill_proficiencies` (IN array)
   - Note: Shows AVAILABLE skills, not guaranteed proficiencies

5. **tool_proficiencies**
   - Type: Multiselect
   - Options: Common tools (Thieves' Tools, etc.)
   - Field: `tool_proficiencies` (IN array)
   - Use: Niche, only 2 results for Thieves' Tools

#### FIX: None

#### REMOVE: None

#### BLOCKED (investigate before implementing):
- **saving_throw_proficiencies**
  - Reason: API test returned 0 results (may be backend indexing issue)
  - Action: Verify with backend team if field is indexed in Meilisearch
  - Priority: Would be HIGH if working (critical for multiclass)

---

## 5. BACKGROUNDS Page Action Plan

### Current State
- **Filters Implemented:** 1 (source_codes only)
- **Status:** CRITICAL - Only 25% complete!
- **Previous Audit Error:** Claimed "100% complete" but was wrong

### Actions Required

#### ADD (3 HIGH priority filters):
1. **skill_proficiencies** (18 skills)
   - Type: Multiselect
   - Options: All 18 skills from `/api/v1/skills` endpoint
   - Field: `skill_proficiencies` (IN array, Meilisearch indexed)
   - Use: "I need Stealth" → 4 backgrounds (Charlatan, Criminal, Feylost, Spy)
   - Implementation: Use `useReferenceData('/skills')`
   - Tested: `athletics` → 7 backgrounds, `deception` → 4 backgrounds

2. **tool_proficiency_types** (3 types)
   - Type: Multiselect
   - Options: Artisan Tools, Musical Instruments, Gaming Sets
   - Field: `tool_proficiency_types` (IN array, Meilisearch indexed)
   - Use: "I need artisan's tools" → 4 backgrounds
   - Tested: `artisan` → 4, `musical` → 5, `gaming` → 9

3. **grants_language_choice** (14 backgrounds)
   - Type: Boolean toggle (Yes/No/All)
   - Field: `grants_language_choice` (boolean, Meilisearch indexed)
   - Use: "Which backgrounds give me extra languages?"
   - Tested: `true` → 14 backgrounds

#### FIX: None

#### REMOVE: None

---

## 6. FEATS Page Action Plan

### Current State
- **Filters Implemented:** 5 user-facing filters
- **Status:** ✅ 100% COMPLETE

### Actions Required

#### ADD: None

#### FIX: None

#### REMOVE: None

#### NOTES:
- `tag_slugs` field exists in API but has NO DATA (backend limitation)
- All practical user filters are implemented
- `id` and `slug` filters exist in API but are not user-facing

---

## Implementation Priority Matrix

| Priority | Entity | Task Type | Filters | Estimated Time | Impact |
|----------|--------|-----------|---------|----------------|--------|
| **1 - CRITICAL** | Monsters | FIX chips | 5 | 30 min | UX consistency |
| **2 - HIGH** | Backgrounds | ADD filters | 3 | 2-3 hours | 300% increase |
| **3 - HIGH** | Items | ADD filters | 6 | 2-3 hours | Core shopping |
| **3 - HIGH** | Items | REMOVE broken | 1 | 10 min | Clean up |
| **4 - MEDIUM** | Races | ADD filter | 1 | 30 min | Easy win |
| **5 - MEDIUM** | Monsters | ADD filters | 6 | 1.5-2 hours | DM tools |
| **6 - MEDIUM** | Classes | ADD filters | 5 | 2-3 hours | Multiclass |

**Total Estimated Time:** 9-12 hours for all actions

---

## Summary by Action Type

### FIX (Critical Bugs)
- **Monsters:** 5 missing chips (30 min)
- **Total:** 5 bugs

### ADD (New Filters)
- **Items:** 6 filters
- **Monsters:** 6 filters
- **Races:** 1 filter
- **Classes:** 5 filters
- **Backgrounds:** 3 filters
- **Total:** 21 new filters

### REMOVE (Broken Features)
- **Items:** 1 broken filter (has_prerequisites)
- **Total:** 1 removal

### Grand Total: 27 actions across 6 entities

---

**Files Referenced:**
- Master summary: `docs/plans/2025-11-25-re-audit-summary.md`
- Monster plan: `docs/plans/2025-11-25-monsters-fixes-and-additions.md`
- Backgrounds plan: `docs/plans/2025-11-25-backgrounds-complete-filters.md` (to be created)
- Items plan: `docs/plans/2025-11-25-items-weapon-armor-filters.md` (to be created)
- Races plan: `docs/plans/2025-11-25-races-parent-filter.md` (to be created)
- Classes plan: `docs/plans/2025-11-25-classes-advanced-filters.md` (to be created)
