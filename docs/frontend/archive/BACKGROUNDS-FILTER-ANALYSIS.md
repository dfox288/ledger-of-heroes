# Backgrounds Filter Analysis & Implementation Report

**Date:** 2025-11-25 (Updated: 2025-11-25 - Comprehensive Audit)
**Agent:** Claude Code
**Task:** Comprehensive analysis and implementation of ALL available filters for Backgrounds entity

---

## Executive Summary

**IMPLEMENTED:** 1 filter (Source)
**API LIMITATIONS:** Backgrounds have minimal filtering capabilities compared to other entities
**STATUS:** ✅ Complete - All available filters implemented
**LAST AUDIT:** 2025-11-25 - No changes to API since initial implementation

---

## API Analysis

### Available Filterable Fields (Per API Documentation)

According to `/v1/backgrounds` endpoint filter parameter:

```
filter: "Meilisearch filter expression for advanced filtering.
         Supports operators: =, !=, AND, OR, IN.
         Available fields: id (int), slug (string), source_codes (array), tag_slugs (array)."
```

**Example:** `tag_slugs IN [criminal, noble]`

### Data Analysis

1. **Total Backgrounds:** 34
2. **Sources Used:**
   - PHB (Player's Handbook) - 32 backgrounds
   - ERLW (Eberron: Rising from the Last War) - 1 background
   - WGTE (Wayfinder's Guide to Eberron) - 1 background
3. **Tags:** Empty dataset - no backgrounds have tags in current data
4. **Proficiencies:** Available in detail view but NOT filterable via Meilisearch
5. **Skills:** Available in detail view but NOT filterable via Meilisearch

---

## Implemented Filters

### 1. Source Filter (Multiselect)

**Type:** `UiFilterMultiSelect`
**Field:** `source_codes`
**Values:** PHB, ERLW, WGTE
**Query:** `source_codes IN [PHB, ERLW]`

**Implementation:**
- Uses `useReferenceData<Source>` to fetch sources
- Filters to only show sources used by backgrounds
- Displays full source names (e.g., "Player's Handbook")
- Stores as codes (e.g., "PHB")

**Tests:** 13 tests covering:
- UI rendering when filters open
- Filter state management
- Badge count updates
- Active filter chips
- Meilisearch query generation
- Clear filters functionality

---

## Filters NOT Implementable (API Limitations)

### Skills/Proficiencies

**Why NOT available:**
- Skills are stored in related `proficiencies` table
- NOT indexed in Meilisearch for backgrounds endpoint
- NOT exposed in filter documentation

**What would be useful:**
```
# DESIRED but not supported:
?filter=skill_slugs IN [stealth, deception]
?filter=proficiency_type = tool
```

**Current data shows:**
- Acolyte: Insight, Religion
- Criminal: Deception, Stealth
- Entertainer: Acrobatics, Performance
- Noble: History, Persuasion
- etc.

**Impact:** Users can't filter "backgrounds with Stealth proficiency" or "backgrounds with tool proficiencies"

### Languages

**Why NOT available:**
- Languages granted by backgrounds vary (0, 1, 2, or choice)
- NOT indexed in Meilisearch
- Complex structure (some grant specific languages, others grant "any 2")

**What would be useful:**
```
# DESIRED but not supported:
?filter=languages_count >= 2
?filter=grants_language_choice = true
```

### Equipment/Starting Gold

**Why NOT available:**
- Equipment is text description in traits, not structured data
- No GP value calculated or indexed
- NOT exposed in filter parameters

### Theme/Social Status/Profession

**Why NOT available:**
- No tagging system in place for backgrounds
- `tag_slugs` field exists but has NO data in any of the 34 backgrounds
- Would require backend team to add tags like:
  - `criminal`, `noble`, `religious`, `martial`, `arcane`
  - `urban`, `rural`, `outlander`
  - `commoner`, `aristocrat`, `soldier`, `entertainer`

**What would be useful:**
```
# DESIRED but not supported (tags empty):
?filter=tag_slugs IN [criminal, noble]  # API supports syntax, but no data
?filter=tag_slugs IN [urban, rural]
```

### Feature Type

**Why NOT available:**
- Background features are text in traits table
- No categorization or tagging
- NOT indexed

**What would be useful:**
```
# DESIRED but not supported:
?filter=feature_type IN [social, utility, roleplay]
```

---

## Comparison with Other Entities

### Spells (10 filters)
- Level (multiselect)
- School (dropdown)
- Class (dropdown)
- Concentration (toggle)
- Ritual (toggle)
- Damage Types (multiselect)
- Saving Throws (multiselect)
- Sources (multiselect)
- Tags (multiselect)
- Components (3 toggles)

### Items (6+ filters)
- Item Type (dropdown)
- Rarity (multiselect)
- Attunement (toggle)
- Magic (toggle)
- Sources (multiselect)
- Tags (multiselect)

### Backgrounds (1 filter) ⚠️
- Sources (multiselect)
- ~~Tags (no data)~~
- ~~Skills (not indexed)~~
- ~~Proficiencies (not indexed)~~

**Backgrounds have the FEWEST filters of all entities.**

---

## Recommendations for Backend Team

### Priority 1: Add Tag System

**Implementation:**
1. Create background tags:
   - **Theme:** `criminal`, `noble`, `religious`, `martial`, `arcane`, `scholarly`, `outdoorsman`
   - **Social Status:** `commoner`, `aristocrat`, `outlander`, `urban-dweller`
   - **Profession:** `soldier`, `entertainer`, `craftsman`, `sailor`, `criminal`, `scholar`
2. Tag existing 34 backgrounds
3. Tags already filterable via `tag_slugs IN [...]` - just need data!

**Examples:**
- Acolyte → `religious`, `scholarly`
- Criminal → `criminal`, `urban-dweller`
- Entertainer → `entertainer`, `performer`, `urban-dweller`
- Folk Hero → `commoner`, `rural`, `protector`
- Noble → `aristocrat`, `noble`, `urban-dweller`
- Outlander → `outlander`, `rural`, `survivalist`
- Sage → `scholarly`, `arcane`, `urban-dweller`
- Soldier → `martial`, `soldier`, `disciplined`

### Priority 2: Index Skills in Meilisearch

**Implementation:**
1. Add `skill_slugs` array to backgrounds Meilisearch index
2. Extract from proficiencies relationship
3. Example: `["insight", "religion"]` for Acolyte
4. Enable filter: `skill_slugs IN [stealth, deception]`

**Use case:**
- "Show backgrounds that give Stealth" (Criminal, Urchin)
- "Show backgrounds that give social skills" (Persuasion, Deception, Performance)

### Priority 3: Index Proficiency Types

**Implementation:**
1. Add `proficiency_types` array to Meilisearch index
2. Values: `skill`, `tool`, `language`
3. Add `tool_count`, `language_count` integers
4. Enable filters:
   - `proficiency_types IN [tool]` (has any tool proficiency)
   - `tool_count >= 1`
   - `language_count >= 2`

**Use case:**
- "Show backgrounds with tool proficiencies" (Criminal, Guild Artisan)
- "Show backgrounds with 2+ languages" (Acolyte)

### Priority 4: Structured Equipment Data

**Implementation:**
1. Calculate starting equipment GP value
2. Add `equipment_value` integer to index
3. Enable filter: `equipment_value >= 15`

**Use case:**
- "Show backgrounds with valuable starting equipment"
- Less critical than skills/proficiencies

---

## Current User Experience

### What Users CAN Do ✅
- Filter by source book (PHB, ERLW, WGTE)
- Search by name (full-text)
- Browse all 34 backgrounds

### What Users CANNOT Do ❌
- Filter by skill proficiencies (Stealth, Insight, etc.)
- Filter by proficiency type (tools, languages)
- Filter by theme (criminal, noble, religious, etc.)
- Filter by social status (commoner, aristocrat, outlander)
- Filter by feature type (social utility, exploration, etc.)

### Compared to Spells Page
- Spells: 10 filters, highly specific searches
- Backgrounds: 1 filter, mostly browse-only

---

## Implementation Details

### Files Modified

1. **`/app/pages/backgrounds/index.vue`**
   - Added source filter state (`selectedSources`)
   - Added `useReferenceData` for sources
   - Added `useMeilisearchFilters` composable
   - Added `useFilterCount` for badge
   - Added source multiselect UI
   - Added active filter chip for source
   - Updated clear filters logic

2. **`/tests/pages/backgrounds-source-filter.test.ts`** (NEW)
   - 13 tests covering all functionality
   - TDD approach (RED → GREEN → REFACTOR)
   - Tests filter UI, state, chips, queries, clear

### Pattern Followed

✅ Follows Spells page pattern exactly:
- Uses `useMeilisearchFilters` composable
- Uses `useReferenceData` for reference data
- Uses `useFilterCount` for badge
- Uses `UiFilterMultiSelect` component
- Active filter chips with clear buttons
- Same layout and structure

### Code Quality

✅ **TDD Mandate:** Tests written first, watched fail, then implemented
✅ **Type Safety:** Full TypeScript with proper types
✅ **Composables:** Reuses existing filter composables
✅ **Components:** Reuses existing UI components
✅ **Testing:** 13 tests, all passing
✅ **Documentation:** This comprehensive report

---

## Testing

### Test Results

```
✓ ../tests/pages/backgrounds-source-filter.test.ts (13 tests) 277ms
✓ ../tests/pages/backgrounds/slug.test.ts (8 tests) 313ms
✓ ../tests/pages/backgrounds-filters.test.ts (9 tests) 213ms

Test Files  3 passed (3)
Tests  30 passed (30)
```

### Test Coverage

- [x] Filter UI renders when open
- [x] Source options computed property exists
- [x] Filter state updates correctly
- [x] Supports multiple source selections
- [x] Badge count shows active filters
- [x] Badge counts multiple sources as 1 filter
- [x] Badge includes source filter in count
- [x] Active filter chip displays
- [x] Multiple sources show in chip
- [x] Clicking chip clears filter
- [x] Meilisearch query for single source
- [x] Meilisearch query for multiple sources
- [x] Clear filters clears source

---

## Browser Testing Checklist

- [ ] Page loads without errors (http://localhost:3000/backgrounds)
- [ ] Source filter multiselect opens and displays options
- [ ] Selecting PHB filters to PHB backgrounds only
- [ ] Selecting multiple sources shows correct results
- [ ] Active filter chip displays correctly
- [ ] Clicking chip clears filter
- [ ] Badge count shows correct number
- [ ] Clear filters button works
- [ ] Dark mode displays correctly
- [ ] Mobile responsive (375px, 768px)
- [ ] Filter panel collapses/expands smoothly

---

## Summary

### What Was Implemented ✅

1. **Source Filter** - Multiselect filter for PHB, ERLW, WGTE
2. **Filter Badge** - Shows count of active filters
3. **Active Chips** - Removable chips for active filters
4. **Clear Filters** - Clears all filters at once
5. **Tests** - 13 comprehensive tests, all passing

### What API Doesn't Support ❌

1. **Skills** - Not indexed (Stealth, Insight, etc.)
2. **Proficiencies** - Not indexed (tools, languages)
3. **Tags** - Field exists but NO data in dataset
4. **Equipment Value** - Not calculated or indexed
5. **Theme/Type** - No categorization system

### Key Finding 🔍

**Backgrounds have the most limited filtering of all entities** due to:
1. Simple data model (name, description, proficiencies)
2. Most data in unstructured text (traits table)
3. No tagging system implemented
4. Skills/proficiencies not indexed in Meilisearch

### Recommendation 🎯

**This is OK!** Backgrounds are a simpler entity type with only 34 total items. Users can:
- Browse all backgrounds easily
- Use search for specific names
- Filter by source if needed

**For future enhancement**, backend team should prioritize:
1. Tag system (criminal, noble, religious, etc.)
2. Index skills in Meilisearch
3. Index proficiency types

---

## Next Steps

1. ✅ Implementation complete
2. ✅ Tests passing
3. ⏳ Browser verification needed
4. ⏳ CHANGELOG update needed
5. ❌ DO NOT COMMIT (per instructions)

---

## 2025-11-25 Audit Results

### API Documentation Re-Check

**Endpoint:** `GET /v1/backgrounds`

**Filter Parameter:**
```
"Meilisearch filter expression for advanced filtering.
Supports operators: =, !=, AND, OR, IN.
Available fields: id (int), slug (string), source_codes (array), tag_slugs (array)."
Example: "tag_slugs IN [criminal, noble]"
```

### Changes Since Initial Implementation

**NO CHANGES DETECTED:**

1. **Total Backgrounds:** Still 34 (unchanged)
2. **Filterable Fields:** Still only `id`, `slug`, `source_codes`, `tag_slugs` (unchanged)
3. **Tag Data:** Still EMPTY - no backgrounds have tags (unchanged)
4. **Proficiencies:** Still available in detail view but NOT indexed for filtering (unchanged)
5. **Skills:** Still available in detail view but NOT indexed for filtering (unchanged)
6. **Equipment:** Still text-based, NOT indexed for filtering (unchanged)

### API Schema Verification

**BackgroundResource Schema Fields:**
- `id` (integer) - Filterable
- `slug` (string) - Filterable
- `name` (string) - Searchable only (full-text)
- `traits` (array) - NOT filterable
- `proficiencies` (array) - NOT filterable
- `sources` (array) - Filterable via `source_codes`
- `languages` (array) - NOT filterable
- `equipment` (array) - NOT filterable
- `tags` (array) - Filterable via `tag_slugs` but NO DATA

### Relationship Endpoints Discovery

**Found in OpenAPI spec but NOT IMPLEMENTED:**
- `/v1/languages/{language}/backgrounds` - 404 Not Found
- `/v1/proficiency-types/{proficiencyType}/backgrounds` - 404 Not Found

These endpoints are documented but do not exist. Backend may have planned them but never implemented.

### Current Filter Implementation Status

| Filter | Status | Notes |
|--------|--------|-------|
| Source (source_codes) | ✅ IMPLEMENTED | Working, all 3 sources (PHB, ERLW, WGTE) |
| Tags (tag_slugs) | ❌ NOT VIABLE | API supports syntax but zero data |
| Skills | ❌ NOT AVAILABLE | Not indexed in Meilisearch |
| Proficiencies | ❌ NOT AVAILABLE | Not indexed in Meilisearch |
| Languages | ❌ NOT AVAILABLE | Not indexed in Meilisearch |
| Equipment | ❌ NOT AVAILABLE | Not indexed in Meilisearch |

### Comparison: Previous Analysis vs Current Audit

| Aspect | Previous Analysis | Current Audit | Change? |
|--------|------------------|---------------|---------|
| Total Backgrounds | 34 | 34 | No |
| Filterable Fields | 4 (id, slug, source_codes, tag_slugs) | 4 (id, slug, source_codes, tag_slugs) | No |
| Tag Data | Empty | Empty | No |
| Implemented Filters | 1 (Source) | 1 (Source) | No |
| API Limitations | Skills, proficiencies, languages not indexed | Skills, proficiencies, languages not indexed | No |
| Potential Filters | 8+ blocked by API | 8+ blocked by API | No |

**VERDICT:** Zero changes to API capabilities. Previous analysis remains 100% accurate.

---

## Conclusion

**Mission Complete** ✅

I've implemented ALL available filters for Backgrounds (1 filter: Source). The API has severe limitations compared to other entities, with only `source_codes` and `tag_slugs` (empty) available for filtering.

This is the most comprehensive implementation possible given API constraints. The codebase is production-ready with 13 passing tests following TDD methodology.

**Backgrounds page: 1 filter implemented, 0 remaining, 8+ potential filters blocked by API limitations.**

**Audit Conclusion (2025-11-25):** No new filterable fields have been added to the API since initial implementation. All recommendations for backend team remain valid.
