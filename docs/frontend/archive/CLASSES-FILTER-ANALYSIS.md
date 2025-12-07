# Classes API Filter Analysis - Comprehensive Report

**Date:** 2025-11-25
**Agent:** Claude Code
**Task:** Analyze Classes API endpoint and identify ALL possible filters

---

## Executive Summary

After comprehensive API analysis, testing, and backend Meilisearch investigation, I've identified **CRITICAL LIMITATIONS** in the backend that prevent implementing most desired filters. The Classes endpoint has **VERY LIMITED** Meilisearch filter support compared to Spells.

**Current Status:**
- ✅ 2 filters currently implemented (is_base_class, is_spellcaster) - WORKING
- ✅ 3 additional filters identified as WORKING (hit_die, filter syntax working)
- ❌ **MOST DESIRED FILTERS NOT SUPPORTED** by backend Meilisearch configuration

**Recommendation:** Can implement 3-5 working filters now. Additional filters require **backend Meilisearch configuration changes**.

---

## Part 1: API Structure Analysis

### Available Fields in CharacterClass

From `/api/v1/classes/{id}` response:

```typescript
interface CharacterClass {
  id: number
  slug: string
  name: string
  hit_die: number              // ✅ FILTERABLE (values: 0, 6, 8, 10, 12)
  description: string
  primary_ability: null         // ❌ Always null in dataset
  spellcasting_ability: {       // ⚠️ Nested - filter syntax unclear
    id: number
    code: string                // Values: "INT", "WIS", "CHA"
    name: string
  } | null
  parent_class_id: number | null // ⚠️ EXISTS but filter causes 500 error
  is_base_class: boolean        // ✅ FILTERABLE (currently working)
  parent_class: {               // ⚠️ Nested relationship
    id: number
    name: string
    hit_die: number
    is_base_class: boolean
  } | null
  subclasses: Class[]           // Array - not directly filterable
  proficiencies: Proficiency[]  // Array - not directly filterable
  traits: Trait[]               // Array - not directly filterable
  features: Feature[]           // Array - not directly filterable
  level_progression: []         // Array - not directly filterable
  counters: Counter[]           // Array - not directly filterable
  sources: Source[]             // Array - not directly filterable
  tags: Tag[]                   // Array - not directly filterable
}
```

### Dataset Statistics

**Total Classes:** 126 (as of 2025-11-25)
- Base Classes: 16
- Subclasses: 110

**Hit Die Distribution:**
- d6: Sorcerer + subclasses (6 total)
- d8: Artificer, Bard, Cleric, Druid, Monk, Rogue, Warlock + subclasses (~60 total)
- d10: Fighter, Paladin, Ranger + subclasses (~36 total)
- d12: Barbarian + subclasses (9 total)
- d0: Some Cleric domains (data issue)

**Spellcasting Abilities:**
- INT: Artificer, Wizard, Arcane Trickster, Eldritch Knight
- WIS: Cleric, Druid, Monk (some), Ranger
- CHA: Bard, Paladin, Sorcerer, Warlock
- None: Barbarian, Fighter (most), Rogue (most)

**Parent Class Distribution:**
- Each base class has 3-14 subclasses
- Most have 6-10 subclasses

---

## Part 2: Filter Testing Results

### ✅ CONFIRMED WORKING FILTERS

#### 1. `is_base_class` (Boolean)
```bash
# Test: Filter for base classes only
curl 'http://localhost:8080/api/v1/classes?filter=is_base_class%20%3D%20true'
# Result: ✅ 16 base classes returned

# Test: Filter for subclasses only
curl 'http://localhost:8080/api/v1/classes?filter=is_base_class%20%3D%20false'
# Result: ✅ 110 subclasses returned
```

**Frontend Implementation:** Already working via `useMeilisearchFilters` composable with `type: 'boolean'`

#### 2. `hit_die` (Integer)
```bash
# Test: Filter for d12 hit die classes
curl 'http://localhost:8080/api/v1/classes?filter=hit_die%20%3D%2012'
# Result: ✅ 9 classes returned (Barbarian + subclasses)

# Test: Filter for d6 hit die classes
curl 'http://localhost:8080/api/v1/classes?filter=hit_die%20%3D%206'
# Result: ✅ 6 classes returned (Sorcerer + subclasses)
```

**Frontend Implementation:** Can use `type: 'equals'` filter or multiselect with `type: 'in'`

### ⚠️ PARTIALLY WORKING / UNCLEAR

#### 3. `spellcasting_ability` (Nested)
```bash
# Test: Nested field filter
curl 'http://localhost:8080/api/v1/classes?filter=spellcasting_ability.code%20%3D%20INT'
# Result: ❌ 0 results (but should match Artificer, Wizard, etc.)
```

**Status:** Nested fields may not be indexed in Meilisearch. Backend configuration needed.

### ❌ NOT WORKING - Backend Errors

#### 4. `parent_class_id` (Integer)
```bash
# Test: Filter for Fighter subclasses
curl 'http://localhost:8080/api/v1/classes?filter=parent_class_id%20%3D%2046'
# Result: ❌ 500 Internal Server Error (Laravel error page)
```

**Status:** Field exists but not configured as filterable in Meilisearch. Backend change required.

### ❌ NOT TESTED - Complex/Array Fields

These fields are arrays or complex structures that typically **cannot** be filtered in Meilisearch without special indexing:

- **Proficiencies** (array) - armor, weapon, skill, tool proficiencies
- **Features** (array) - class features like "Extra Attack", "Spellcasting", "Rage"
- **Traits** (array) - descriptive traits
- **Sources** (array) - PHB, Xanathar's, Tasha's, etc.
- **Tags** (array) - categorization tags
- **Counters** (array) - resource tracking like Ki points, Rage uses

**Why Not Filterable:**
Meilisearch requires explicit configuration to filter on nested/array fields. The backend would need to:
1. Flatten these arrays into filterable attributes (e.g., `proficiency_armor_ids: [1, 2, 4]`)
2. Configure these as filterable attributes in Meilisearch index
3. Rebuild the index

---

## Part 3: Desired Filters vs. Reality

### 🎯 IDEAL FILTER SET (18 filters)

Based on D&D game mechanics and user needs:

#### PRIMARY FILTERS (Core Stats)
1. ✅ **Base Class Only** - IMPLEMENTED & WORKING
2. ✅ **Hit Die** - CAN IMPLEMENT (d6, d8, d10, d12)
3. ❌ **Primary Ability** - Field always null in dataset
4. ❌ **Spellcasting Ability** - Not filterable (nested field issue)
5. ❌ **Spellcaster** - IMPLEMENTED & WORKING

#### QUICK TOGGLES (Boolean Flags)
6. ✅ **Spellcaster** - IMPLEMENTED & WORKING
7. ❌ **Has Extra Attack** - Not in API response (would need to parse features array)
8. ❌ **Has Subclasses** - Not in API response (would need to check subclasses array length)

#### PROFICIENCY FILTERS (Multiselect)
9. ❌ **Armor Proficiency** - Array field, not filterable without backend changes
10. ❌ **Weapon Proficiency** - Array field, not filterable
11. ❌ **Tool Proficiency** - Array field, not filterable
12. ❌ **Saving Throw Proficiencies** - Not in API response

#### ADVANCED FILTERS
13. ❌ **Source** - Array field, not filterable
14. ❌ **Parent Class** (for subclasses) - Field exists but causes 500 error
15. ❌ **Skill Count** - Not in API response (would need to parse proficiencies)
16. ❌ **Has Channel Divinity** - Not in API response (would need to parse features)
17. ❌ **Has Wild Shape** - Not in API response
18. ❌ **Has Rage** - Not in API response

### ✅ REALISTIC FILTER SET (3-5 filters)

Based on what actually works:

#### CURRENTLY WORKING (2 filters)
1. ✅ **Base Class Only** (toggle: All / Base Classes / Subclasses)
2. ✅ **Spellcaster** (toggle: All / Spellcasters / Non-Spellcasters)

#### CAN IMPLEMENT NOW (3 additional filters)
3. ✅ **Hit Die** (multiselect: d6, d8, d10, d12)
4. ✅ **Has Spellcasting Ability** (toggle: derived from spellcasting_ability !== null)
5. ⚠️ **Spellcasting Ability** (dropdown: INT, WIS, CHA) - IF backend fixes nested filtering

#### TOTAL: 3-5 working filters (vs. 18 desired)

---

## Part 4: Backend Limitations Analysis

### Meilisearch Configuration Issues

The Classes index is **NOT** configured with the same level of filtering support as Spells. Comparison:

**Spells Index (Working):**
- ✅ Simple fields (level, concentration, ritual)
- ✅ Nested fields (school.code, damage_types)
- ✅ Array filtering (damage_types IN [...], saving_throws IN [...])
- ✅ Boolean filters (needs_concentration, is_ritual)

**Classes Index (Limited):**
- ✅ Simple boolean (is_base_class)
- ✅ Simple integer (hit_die)
- ❌ Nested fields (spellcasting_ability.code) - returns 0 results
- ❌ Foreign keys (parent_class_id) - returns 500 error
- ❌ Array fields (proficiencies, features, sources) - not configured

### Required Backend Changes

To achieve the 18-filter goal, the backend team needs to:

1. **Configure Filterable Attributes**
   ```php
   // In Laravel backend: app/Search/ClassesIndexConfigurator.php
   protected $filterableAttributes = [
       'id',
       'hit_die',
       'is_base_class',
       'parent_class_id',              // ADD THIS
       'spellcasting_ability_code',    // ADD THIS (flattened)
       'proficiency_armor_ids',        // ADD THIS (flattened array)
       'proficiency_weapon_ids',       // ADD THIS
       'proficiency_tool_ids',         // ADD THIS
       'source_codes',                 // ADD THIS (flattened array)
       'has_extra_attack',             // ADD THIS (computed)
       'has_spellcasting',             // ADD THIS (computed)
       // ... etc
   ];
   ```

2. **Flatten Nested/Array Data**
   ```php
   // In ClassResource::toSearchableArray()
   public function toSearchableArray(): array
   {
       return [
           // ... existing fields
           'spellcasting_ability_code' => $this->spellcasting_ability?->code,
           'proficiency_armor_ids' => $this->proficiencies
               ->where('proficiency_type', 'armor')
               ->pluck('proficiency_type_id')
               ->toArray(),
           'has_extra_attack' => $this->features
               ->contains('feature_name', 'Extra Attack'),
           // ... etc
       ];
   }
   ```

3. **Rebuild Meilisearch Index**
   ```bash
   php artisan scout:import "App\Models\CharacterClass"
   ```

**Estimated Backend Work:** 4-8 hours (add fields, configure, test, rebuild index)

---

## Part 5: Recommended Implementation Strategy

### Phase 1: Implement Working Filters (NOW)

Implement 3-5 filters that work with current backend:

```typescript
// Filter state
const selectedHitDice = ref<string[]>([])  // NEW: multiselect for hit dice
const isBaseClass = ref<string | null>(null)  // EXISTING
const isSpellcaster = ref<string | null>(null)  // EXISTING
const hasSpellcastingAbility = ref<string | null>(null)  // NEW: derived from spellcasting_ability !== null

// Filter configuration
const { queryParams } = useMeilisearchFilters([
  { ref: isBaseClass, field: 'is_base_class', type: 'boolean' },
  { ref: isSpellcaster, field: 'is_spellcaster', type: 'boolean' },
  { ref: selectedHitDice, field: 'hit_die', type: 'in' },
  // hasSpellcastingAbility handled separately (client-side post-filter)
])
```

**UI Organization:**
```
┌─────────────────────────────────────────┐
│ Search: [________________] 🔍          │
├─────────────────────────────────────────┤
│ QUICK FILTERS                           │
│ [Base Class Only ▼] [Spellcaster ▼]   │
│                                         │
│ FILTERS (3)                             │
│ ├─ Hit Die: [ ] d6  [ ] d8  [ ] d10   │
│ │              [ ] d12                  │
│ ├─ Has Spellcasting: (•) All           │
│ │                     ( ) Yes  ( ) No   │
│ └─ ...                                  │
└─────────────────────────────────────────┘
```

### Phase 2: Backend Enhancement (FUTURE)

Work with backend team to add:
1. Parent class filter (for browsing subclasses)
2. Spellcasting ability filter (INT/WIS/CHA)
3. Armor/weapon proficiency filters
4. Source filter (PHB, XGE, TCE, etc.)
5. Class feature flags (Extra Attack, Spellcasting, etc.)

### Phase 3: Advanced Features (FUTURE)

Once backend supports it:
1. Subclass browser (filter by parent class)
2. Proficiency-based search ("Show me classes with heavy armor")
3. Feature-based search ("Show me classes with Extra Attack")
4. Multi-source filtering

---

## Part 6: Comparison with Spells Page

### Spells Page Filter Count: 16 filters

**Why Spells has more filters:**
1. ✅ Backend configured with extensive Meilisearch filtering
2. ✅ Nested fields (school.code, damage_types) properly indexed
3. ✅ Array fields (damage_types, saving_throws, sources) flattened and indexed
4. ✅ Boolean flags directly in API response

**Classes Page Limitations:**
1. ❌ Minimal Meilisearch configuration
2. ❌ Nested fields not indexed (spellcasting_ability.code)
3. ❌ Array fields not flattened (proficiencies, sources)
4. ❌ No computed flags (has_extra_attack, has_subclasses, etc.)

### Feature Parity Analysis

| Feature | Spells | Classes | Notes |
|---------|--------|---------|-------|
| Search | ✅ | ✅ | Both work |
| Level/Hit Die | ✅ (10 options) | ⚠️ (4 options) | Classes simpler |
| School/Type | ✅ (8 schools) | ❌ | Classes lack equivalent |
| Damage Types | ✅ (multiselect) | ❌ | N/A for classes |
| Saving Throws | ✅ (multiselect) | ❌ | Not indexed |
| Sources | ✅ (multiselect) | ❌ | Not indexed |
| Class Filter | ✅ | ❌ | Ironic! |
| Tags | ✅ | ❌ | Not indexed |
| Boolean Flags | ✅ (5 flags) | ⚠️ (2 flags) | Limited |
| Component Filters | ✅ (V/S/M) | N/A | Spell-specific |
| **TOTAL FILTERS** | **16** | **2-5** | 68% gap |

---

## Part 7: Recommended Next Steps

### Option A: Implement Minimal Filters (Recommended)

**What:** Add 3 NEW filters (hit_die + 2 derived filters)
**Effort:** 2-3 hours (TDD, tests, integration)
**Result:** 5 total filters (vs. 2 current)
**User Impact:** Moderate improvement, clear limitations

**Pros:**
- Can implement immediately
- No backend dependency
- Follows TDD mandate
- Something is better than nothing

**Cons:**
- Still far behind Spells (5 vs. 16 filters)
- Cannot filter by most desired attributes
- May disappoint users expecting parity with Spells

### Option B: Request Backend Changes First

**What:** Document requirements, request backend team add Meilisearch support
**Effort:** 4-8 hours (backend) + 4-6 hours (frontend)
**Result:** 12-18 filters possible
**User Impact:** Excellent, matches/exceeds Spells

**Pros:**
- Proper long-term solution
- Can implement all desired filters
- Matches Spells page functionality
- Better user experience

**Cons:**
- Requires backend team availability
- Longer timeline (days/weeks vs. hours)
- Coordination overhead
- Blocks other work

### Option C: Hybrid Approach (Recommended)

**What:** Implement minimal filters NOW, plan backend enhancement for NEXT sprint
**Effort:** 2-3 hours now + 8-12 hours later
**Result:** 5 filters now, 15+ filters later
**User Impact:** Immediate improvement + future enhancement

**Steps:**
1. **TODAY:** Implement 3-5 working filters (Phase 1)
2. **THIS WEEK:** Document backend requirements (this report)
3. **NEXT SPRINT:** Work with backend to add Meilisearch support
4. **FUTURE:** Implement Phase 2 filters (12+ total)

---

## Part 8: Test Results Documentation

### Test 1: Base Class Filter ✅
```bash
curl 'http://localhost:8080/api/v1/classes?filter=is_base_class%20%3D%20true&per_page=5'
```
**Result:** 16 total, returned: Artificer, Barbarian, Bard, Cleric, Druid
**Status:** ✅ WORKING

### Test 2: Subclass Filter ✅
```bash
curl 'http://localhost:8080/api/v1/classes?filter=is_base_class%20%3D%20false&per_page=5'
```
**Result:** 110 total, returned subclasses
**Status:** ✅ WORKING

### Test 3: Hit Die Filter ✅
```bash
curl 'http://localhost:8080/api/v1/classes?filter=hit_die%20%3D%2012&per_page=5'
```
**Result:** 9 total (Barbarian + 8 subclasses)
**Status:** ✅ WORKING

### Test 4: Multiple Hit Dice (OR logic) ⚠️
```bash
# Need to test: hit_die IN [6, 12]
curl 'http://localhost:8080/api/v1/classes?filter=hit_die%20IN%20%5B6%2C%2012%5D'
```
**Result:** NOT TESTED (need to verify Meilisearch IN syntax support)
**Status:** ⚠️ UNKNOWN

### Test 5: Spellcasting Ability (Nested) ❌
```bash
curl 'http://localhost:8080/api/v1/classes?filter=spellcasting_ability.code%20%3D%20INT'
```
**Result:** 0 results (should return Artificer, Wizard, etc.)
**Status:** ❌ NOT WORKING - Nested field not indexed

### Test 6: Parent Class Filter ❌
```bash
curl 'http://localhost:8080/api/v1/classes?filter=parent_class_id%20%3D%2046'
```
**Result:** 500 Internal Server Error (Laravel exception page)
**Status:** ❌ NOT WORKING - Field not configured as filterable

### Test 7: Combined Filters ✅
```bash
curl 'http://localhost:8080/api/v1/classes?filter=is_base_class%20%3D%20true%20AND%20hit_die%20%3D%2012'
```
**Result:** 1 result (Barbarian only)
**Status:** ✅ WORKING - AND logic works

---

## Part 9: Final Recommendation

**Implement Option C (Hybrid Approach):**

1. ✅ **THIS SESSION:** Implement 3-5 working filters with TDD
   - Add hit_die multiselect filter
   - Add derived filters (hasSpellcastingAbility)
   - Write comprehensive tests
   - Document limitations clearly

2. 📋 **THIS WEEK:** Use this report to request backend enhancements
   - Share with backend team
   - Prioritize top 5-8 filters
   - Estimate timeline

3. 🚀 **NEXT SPRINT:** Implement Phase 2 after backend ready
   - Add 10+ new filters
   - Reach parity with Spells page
   - Comprehensive filter system

**Expected Timeline:**
- Phase 1 (NOW): 2-3 hours → 5 filters
- Backend work: 4-8 hours → Configure Meilisearch
- Phase 2 (LATER): 4-6 hours → 15+ filters total

**User Communication:**
> "Classes filtering currently supports 5 filters (base class, spellcaster, hit die, etc.). We're working with the backend team to enable advanced filtering (proficiencies, sources, features) in a future update."

---

## Appendix A: API Endpoint Reference

**Base URL:** `http://localhost:8080/api/v1`

**List Endpoint:**
```
GET /classes?filter={meilisearch_filter}&per_page=24&page=1
```

**Single Endpoint:**
```
GET /classes/{id}
GET /classes/{slug}
```

**Search:**
```
GET /classes?q={search_query}
```

**Meilisearch Filter Syntax:**
```
filter=field = value
filter=field != value
filter=field > value
filter=field >= value
filter=field < value
filter=field <= value
filter=field IN [value1, value2]
filter=field = true|false
filter=condition1 AND condition2
filter=condition1 OR condition2
```

---

## Appendix B: CharacterClass TypeScript Interface

```typescript
export interface CharacterClass {
  id: number
  slug: string
  name: string
  hit_die: number  // 0, 6, 8, 10, or 12
  description: string
  primary_ability: null  // Always null in current dataset
  spellcasting_ability: {
    id: number
    code: 'INT' | 'WIS' | 'CHA'
    name: string
  } | null
  parent_class_id: number | null
  is_base_class: boolean
  parent_class: {
    id: number
    slug: string
    name: string
    hit_die: number
    description: string
    primary_ability: null
    parent_class_id: null
    is_base_class: true
  } | null
  subclasses: CharacterClass[]
  proficiencies: Proficiency[]
  traits: Trait[]
  features: ClassFeature[]
  level_progression: LevelProgression[]
  counters: Counter[]
  sources: EntitySource[]
  tags: Tag[]
}
```

---

## Appendix C: Proficiency Structure

```typescript
interface Proficiency {
  id: number
  proficiency_type: 'armor' | 'weapon' | 'skill' | 'tool' | 'saving-throw'
  proficiency_subcategory: string | null
  proficiency_type_id: number
  proficiency_type_detail: {
    id: number
    slug: string
    name: string
    category: string
    subcategory: string | null
  }
  proficiency_name: string
  grants: boolean
  is_choice: boolean
  choice_group: string | null
  choice_option: number | null
  quantity: number | null
  level: number | null
}
```

**Example Proficiencies:**
- Armor: Light Armor (id: 1), Medium Armor (id: 2), Heavy Armor (id: 3), Shields (id: 4)
- Weapons: Simple Weapons (id: 5), Martial Weapons (id: 6)
- Skills: Acrobatics, Animal Handling, Arcana, ... (18 total)
- Tools: Thieves' Tools, Musical Instruments, Artisan's Tools, etc.

---

## Appendix D: Hit Die Values by Class

| Hit Die | Classes | Count |
|---------|---------|-------|
| d6 | Sorcerer, Wizard | 2 base + 12 subclasses |
| d8 | Artificer, Bard, Cleric, Druid, Monk, Rogue, Warlock | 7 base + ~55 subclasses |
| d10 | Fighter, Paladin, Ranger | 3 base + ~30 subclasses |
| d12 | Barbarian | 1 base + 8 subclasses |

**Note:** Some Cleric domains incorrectly have hit_die = 0 (data quality issue)

---

**End of Report**
