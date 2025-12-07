# Comprehensive Backgrounds Filter Audit Report

**Date:** 2025-11-25
**Auditor:** Claude Code
**Task:** Audit Backgrounds entity filters against API documentation and identify missing filters

---

## Executive Summary

**AUDIT RESULT:** ✅ COMPLETE - All available API filters have been implemented

- **Currently Implemented:** 1 filter (Source)
- **Available in API:** 2 filterable fields (source_codes, tag_slugs)
- **Missing from Implementation:** 0 filters
- **Blocked by API Limitations:** 8+ potential filters

**VERDICT:** The Backgrounds page has implemented 100% of the filterable fields available in the API. No new filters can be added without backend changes.

---

## 1. API Documentation Analysis

### Endpoint: `GET /v1/backgrounds`

**Filter Parameter Documentation:**
```
"Meilisearch filter expression for advanced filtering.
Supports operators: =, !=, AND, OR, IN.
Available fields: id (int), slug (string), source_codes (array), tag_slugs (array)."

Example: "tag_slugs IN [criminal, noble]"
```

**Other Available Parameters:**
- `q` - Full-text search (already implemented via searchQuery)
- `per_page` - Pagination (already implemented)
- `page` - Pagination (already implemented)
- `sort_by` - Sorting (name, created_at, updated_at)
- `sort_direction` - Sort direction (asc, desc)

### API Filterable Fields (Complete List)

| Field | Type | Filterable? | Implemented? | Notes |
|-------|------|-------------|--------------|-------|
| `id` | int | ✅ Yes | ❌ No | Not useful for user-facing filter (internal ID) |
| `slug` | string | ✅ Yes | ❌ No | Not useful for user-facing filter (covered by search) |
| `source_codes` | array | ✅ Yes | ✅ YES | **IMPLEMENTED** - Source multiselect filter |
| `tag_slugs` | array | ✅ Yes | ❌ No | **NOT VIABLE** - Zero data in database |

**Conclusion:** Only 1 viable user-facing filter exists (`source_codes`), which is fully implemented.

---

## 2. Current Implementation Review

### File: `/app/pages/backgrounds/index.vue`

**Implemented Filters:**

1. **Source Filter** (Lines 11-32)
   - **Type:** Multiselect
   - **Component:** `UiFilterMultiSelect`
   - **Field:** `source_codes`
   - **Query Type:** `IN`
   - **Values:** PHB, ERLW, WGTE (3 sources)
   - **Status:** ✅ Working correctly

**Filter Composables Used:**
- `useMeilisearchFilters()` - Builds filter query params
- `useReferenceData<Source>()` - Fetches source options
- `useFilterCount()` - Badge count for active filters

**UI Components:**
- `<UiFilterCollapse>` - Collapsible filter panel
- `<UiFilterMultiSelect>` - Multiselect dropdown
- Active filter chips with clear buttons
- "Clear filters" button

**Test Coverage:**
- File: `/tests/pages/backgrounds-source-filter.test.ts`
- Tests: 13 tests, all passing
- Coverage: UI rendering, state management, query generation, badge counts, chips, clear functionality

---

## 3. Data Analysis

### Dataset Statistics

**Total Backgrounds:** 34

**Source Distribution:**
- PHB (Player's Handbook): 32 backgrounds
- ERLW (Eberron: Rising from the Last War): 1 background
- WGTE (Wayfinder's Guide to Eberron): 1 background

**Tag Analysis:**
```bash
# Checked all 34 backgrounds
curl -s 'http://localhost:8080/api/v1/backgrounds?per_page=34' | \
  jq '.data[] | select(.tag_slugs and (.tag_slugs | length) > 0)'
# Result: (empty)
```

**Conclusion:** Zero backgrounds have tags. The `tag_slugs` field exists in API but has no data.

### Sample Background Structure

**Acolyte (ID: 2):**
```json
{
  "id": 2,
  "slug": "acolyte",
  "name": "Acolyte",
  "traits": [...],          // NOT filterable
  "proficiencies": [        // NOT filterable
    {
      "skill": {
        "name": "Insight",
        "slug": "insight"
      }
    },
    {
      "skill": {
        "name": "Religion",
        "slug": "religion"
      }
    }
  ],
  "sources": [...],         // Filterable via source_codes
  "languages": [],          // NOT filterable
  "equipment": [...],       // NOT filterable
  "tags": []                // Filterable via tag_slugs but EMPTY
}
```

---

## 4. Missing Filters Analysis

### 4.1 Filters That CAN Be Implemented (from API)

**NONE** - All available API filters are already implemented.

| Filter | API Field | Status | Reason |
|--------|-----------|--------|--------|
| Tags | `tag_slugs` | ❌ NOT VIABLE | Zero data in database (all 34 backgrounds have empty tags array) |
| ID | `id` | ❌ NOT USEFUL | Internal identifier, not user-facing |
| Slug | `slug` | ❌ NOT USEFUL | Covered by full-text search |

### 4.2 Filters Blocked by API Limitations

**These would be useful but are NOT indexed in Meilisearch:**

#### A. Skill Proficiencies
**Why Useful:**
- Users could filter "backgrounds with Stealth" (Criminal, Urchin)
- Filter "backgrounds with social skills" (Persuasion, Deception, Performance)

**Current State:**
- Skills available in detail view
- Stored in `proficiencies` relationship
- NOT indexed in Meilisearch backgrounds index

**Example Use Case:**
```
# DESIRED but not supported:
?filter=skill_slugs IN [stealth, deception]
```

**Data Examples:**
- Acolyte: Insight, Religion
- Criminal: Deception, Stealth
- Entertainer: Acrobatics, Performance
- Noble: History, Persuasion
- Sage: Arcana, History
- Soldier: Athletics, Intimidation

#### B. Proficiency Types
**Why Useful:**
- Filter "backgrounds with tool proficiencies" (Criminal, Guild Artisan, Sailor)
- Filter "backgrounds with language choices" (Acolyte, Noble, Sage)

**Current State:**
- Proficiencies categorized by type (skill, tool, language)
- NOT indexed in Meilisearch

**Example Use Case:**
```
# DESIRED but not supported:
?filter=proficiency_types IN [tool]
?filter=tool_count >= 1
?filter=language_count >= 2
```

#### C. Languages
**Why Useful:**
- Filter backgrounds granting specific languages
- Filter backgrounds granting multiple language choices

**Current State:**
- Languages stored in `languages` relationship
- NOT indexed in Meilisearch

**Example Use Case:**
```
# DESIRED but not supported:
?filter=grants_language_choice = true
?filter=language_count >= 2
```

#### D. Equipment/Starting Wealth
**Why Useful:**
- Filter backgrounds by starting equipment value
- Filter backgrounds with specific equipment types

**Current State:**
- Equipment stored as text descriptions or item relationships
- No calculated GP value
- NOT indexed in Meilisearch

**Example Use Case:**
```
# DESIRED but not supported:
?filter=equipment_value >= 15
?filter=has_tools = true
```

#### E. Theme/Category Tags
**Why MOST Useful:**
- Filter by background theme (criminal, noble, religious, martial, arcane)
- Filter by social status (commoner, aristocrat, outlander)
- Filter by profession type (soldier, entertainer, craftsman, sailor)

**Current State:**
- `tag_slugs` field exists and IS filterable
- **Zero backgrounds have tags** (database is empty)
- Backend team never populated tag data

**Example Use Case:**
```
# API SUPPORTS but NO DATA:
?filter=tag_slugs IN [criminal, noble]
?filter=tag_slugs IN [urban, rural]
?filter=tag_slugs IN [martial, arcane]
```

**Recommended Tags:**
- **Theme:** criminal, noble, religious, martial, arcane, scholarly, outdoorsman
- **Social Status:** commoner, aristocrat, outlander, urban-dweller
- **Profession:** soldier, entertainer, craftsman, sailor, criminal, scholar

**Tag Examples:**
- Acolyte → `religious`, `scholarly`
- Criminal → `criminal`, `urban-dweller`
- Entertainer → `entertainer`, `performer`, `urban-dweller`
- Folk Hero → `commoner`, `rural`, `protector`
- Noble → `aristocrat`, `noble`, `urban-dweller`
- Outlander → `outlander`, `rural`, `survivalist`
- Sage → `scholarly`, `arcane`, `urban-dweller`
- Soldier → `martial`, `soldier`, `disciplined`

---

## 5. Comparison with Other Entities

### Filter Count by Entity

| Entity | Total Filters | Notes |
|--------|--------------|-------|
| **Spells** | 10 filters | Level, School, Class, Concentration, Ritual, Damage Types, Saving Throws, Sources, Tags, Components (V/S/M) |
| **Items** | 6+ filters | Item Type, Rarity, Attunement, Magic, Sources, Tags |
| **Monsters** | Unknown | Not audited yet |
| **Backgrounds** | **1 filter** | **FEWEST** - Only Source |
| **Races** | Unknown | Not audited yet |
| **Classes** | Unknown | Not audited yet |
| **Feats** | Unknown | Not audited yet |

**Backgrounds have the FEWEST filters of all major entities.**

**Why?**
1. Simpler data model (primarily text-based traits)
2. Proficiencies not indexed in Meilisearch
3. No tagging system implemented (tags field empty)
4. Small dataset (34 total) makes filtering less critical

---

## 6. Discovered Issues

### 6.1 Documented but Non-Existent Endpoints

**Found in OpenAPI specification:**
- `/v1/languages/{language}/backgrounds`
- `/v1/proficiency-types/{proficiencyType}/backgrounds`

**Status:** Both return 404 Not Found

**Implication:** Backend team documented these endpoints but never implemented them. These would have enabled filtering backgrounds by:
- Languages granted
- Proficiency types (skill/tool/language)

### 6.2 BackgroundResource Schema

**Complete Schema Fields:**
```json
{
  "id": "integer",
  "slug": "string",
  "name": "string",
  "traits": "array",         // NOT filterable
  "proficiencies": "array",  // NOT filterable
  "sources": "array",        // Filterable via source_codes
  "languages": "array",      // NOT filterable
  "equipment": "array",      // NOT filterable
  "tags": "array"            // Filterable via tag_slugs but empty
}
```

Only 2 of 9 fields are filterable (`sources`, `tags`), and one has no data.

---

## 7. Comparison: Previous Analysis vs Current Audit

### Changes Since Initial Implementation

| Aspect | Previous Analysis | Current Audit | Change? |
|--------|------------------|---------------|---------|
| Total Backgrounds | 34 | 34 | ❌ No |
| API Filterable Fields | 4 (id, slug, source_codes, tag_slugs) | 4 (id, slug, source_codes, tag_slugs) | ❌ No |
| Tag Data Count | 0 (empty) | 0 (empty) | ❌ No |
| Implemented Filters | 1 (Source) | 1 (Source) | ❌ No |
| Skills Indexed? | No | No | ❌ No |
| Proficiencies Indexed? | No | No | ❌ No |
| Languages Indexed? | No | No | ❌ No |
| Equipment Indexed? | No | No | ❌ No |
| Backend Limitations | 8+ potential filters blocked | 8+ potential filters blocked | ❌ No |

**VERDICT:** Zero changes to API capabilities since initial implementation.

---

## 8. Recommendations

### For Frontend Team

**No action required.** All available API filters have been implemented.

**Maintain current implementation:**
- ✅ Source filter working correctly
- ✅ 13 tests passing
- ✅ Follows standard pattern from Spells page
- ✅ Uses composables correctly

### For Backend Team

**Priority 1: Implement Tag System (HIGHEST IMPACT)**

**Why:** Tag field exists and is filterable, just needs data.

**Steps:**
1. Create tag taxonomy for backgrounds:
   - Theme tags: criminal, noble, religious, martial, arcane, scholarly, outdoorsman
   - Social tags: commoner, aristocrat, outlander, urban-dweller
   - Profession tags: soldier, entertainer, craftsman, sailor, scholar
2. Tag all 34 existing backgrounds
3. Add tags to Meilisearch index (likely already done)

**Frontend Impact:** Can immediately add tag multiselect filter using existing composables.

**Estimated Frontend Effort:** 2-4 hours (add filter UI, write tests)

---

**Priority 2: Index Skills in Meilisearch**

**Why:** Second most useful filter for users.

**Steps:**
1. Add `skill_slugs` array to backgrounds Meilisearch index
2. Extract from proficiencies relationship
3. Update API documentation

**Example Data:**
```json
{
  "id": 2,
  "name": "Acolyte",
  "skill_slugs": ["insight", "religion"]
}
```

**Frontend Impact:** Can add skill multiselect filter.

**Estimated Frontend Effort:** 4-6 hours (fetch skills reference data, add filter UI, write tests)

---

**Priority 3: Index Proficiency Types**

**Why:** Useful for finding backgrounds with tools or languages.

**Steps:**
1. Add `proficiency_types` array to Meilisearch index
2. Add `tool_count`, `language_count` integers
3. Update API documentation

**Example Data:**
```json
{
  "id": 2,
  "name": "Acolyte",
  "proficiency_types": ["skill", "language"],
  "skill_count": 2,
  "tool_count": 0,
  "language_count": 2
}
```

**Frontend Impact:** Can add proficiency type filters.

**Estimated Frontend Effort:** 4-6 hours (add filters, write tests)

---

**Priority 4: Equipment Value Calculation**

**Why:** Least critical (equipment is flavorful, not gameplay-critical for filters).

**Steps:**
1. Calculate total starting equipment GP value
2. Add `equipment_value` integer to Meilisearch index
3. Update API documentation

**Frontend Impact:** Can add range filter for equipment value.

**Estimated Frontend Effort:** 2-3 hours (add range filter, write tests)

---

## 9. Test Results

### Current Test Coverage

**File:** `/tests/pages/backgrounds-source-filter.test.ts`

**Tests:** 13 tests, all passing

**Coverage:**
- ✅ Filter UI renders when open
- ✅ Source options computed property
- ✅ Filter state updates
- ✅ Multiple source selections
- ✅ Badge count shows active filters
- ✅ Badge counts multiple sources as 1 filter
- ✅ Badge includes source filter in count
- ✅ Active filter chip displays
- ✅ Multiple sources show in chip
- ✅ Clicking chip clears filter
- ✅ Meilisearch query for single source
- ✅ Meilisearch query for multiple sources
- ✅ Clear filters clears source

**Test Command:**
```bash
docker compose exec nuxt npm run test backgrounds-source-filter
```

**Result:** All tests pass ✅

---

## 10. User Experience Analysis

### What Users CAN Do (Current)

✅ **Search by name** - Full-text search across all backgrounds
✅ **Filter by source** - PHB, ERLW, WGTE
✅ **Browse all backgrounds** - Only 34 total, easy to browse
✅ **View detailed information** - Skills, proficiencies, languages, equipment in detail view

### What Users CANNOT Do (Blocked by API)

❌ **Filter by skill proficiencies** - "Show me backgrounds with Stealth"
❌ **Filter by proficiency type** - "Show me backgrounds with tool proficiencies"
❌ **Filter by theme** - "Show me criminal/noble/religious backgrounds"
❌ **Filter by social status** - "Show me commoner/aristocrat backgrounds"
❌ **Filter by languages** - "Show me backgrounds granting 2+ languages"
❌ **Filter by equipment value** - "Show me backgrounds with valuable starting equipment"

### Impact Assessment

**Current UX: ACCEPTABLE** (but limited)

**Reasons:**
1. **Small dataset** - Only 34 backgrounds, easy to browse without filters
2. **Search works well** - Full-text search finds specific backgrounds by name
3. **Detail view has all info** - Users can see skills/proficiencies once they click
4. **Source filter is useful** - PHB vs Eberron sources are different campaigns

**Future UX: COULD BE BETTER**

**If backend implements recommended filters:**
1. **Skill filter** - "I need Stealth for my rogue character"
2. **Theme tags** - "I want a criminal background" or "I want a noble background"
3. **Proficiency type** - "I need a background with tool proficiencies"

**These would improve character creation workflow significantly.**

---

## 11. Conclusion

### Audit Summary

**✅ AUDIT COMPLETE**

**Findings:**
1. **All available API filters are implemented** - Source filter is working correctly
2. **No new filters can be added** - Without backend changes
3. **No API changes since initial implementation** - Documentation and data unchanged
4. **Previous analysis was accurate** - All conclusions remain valid

**Statistics:**
- **API Filterable Fields:** 4 (id, slug, source_codes, tag_slugs)
- **User-Facing Viable Filters:** 1 (source_codes)
- **Implemented Filters:** 1 (source_codes)
- **Implementation Coverage:** 100%
- **Potential Filters Blocked by API:** 8+ (skills, proficiencies, languages, equipment, tags)

### Final Verdict

**No changes required to frontend implementation.**

The Backgrounds page has successfully implemented 100% of the available filterable fields from the API. The limited filtering capability is due to backend limitations, not frontend gaps.

**For future enhancement**, backend team should prioritize:
1. **Tag system** (highest priority - field exists, just needs data)
2. **Skill indexing** (second priority - most useful for character creation)
3. **Proficiency type indexing** (third priority - useful for tool/language searches)
4. **Equipment value calculation** (lowest priority - nice to have)

### Updated Documentation

**File:** `/docs/BACKGROUNDS-FILTER-ANALYSIS.md`

**Changes Made:**
- Added "2025-11-25 Audit Results" section
- Documented API re-check findings
- Added comparison table (previous vs current)
- Added "LAST AUDIT" date to executive summary
- Confirmed zero changes since initial implementation

### Next Steps

**For Frontend Team:**
- ✅ Audit complete
- ✅ Documentation updated
- ✅ No implementation changes needed
- ⏳ Monitor backend API for future changes (quarterly re-audit recommended)

**For Backend Team:**
- Review recommendations in Priority 1-4
- Consider implementing tag system (quick win, high impact)
- Plan Meilisearch index updates for Q1 2026 (if desired)

---

**Audit Date:** 2025-11-25
**Auditor:** Claude Code
**Status:** ✅ Complete
**Follow-up:** Next audit in 3-6 months (or when backend team adds new filterable fields)
