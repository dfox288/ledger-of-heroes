# Spell Filter API Audit - Complete Migration to Meilisearch

**Date:** 2025-11-25
**Purpose:** Audit current spell filters against new API capabilities and plan complete migration to Meilisearch filter syntax
**Status:** 🔴 CRITICAL - Current implementation using deprecated/non-existent MySQL params

---

## 🚨 Critical Finding

**The API has been completely migrated to Meilisearch-only filtering.**

- ❌ **MySQL fallback params NO LONGER EXIST** in OpenAPI spec
- ❌ Current frontend code uses `params.level`, `params.school`, `params.concentration`, `params.ritual`
- ❌ These params are likely being ignored by the API (or will be removed soon)
- ✅ **ALL filters must use `filter` parameter with Meilisearch syntax**

---

## 📊 API Capabilities (New)

### Available Query Parameters

| Parameter | Type | Purpose | Example |
|-----------|------|---------|---------|
| `q` | string | Full-text search | `?q=fireball` |
| `filter` | string | Meilisearch filter expression | `?filter=level = 3 AND concentration = true` |
| `sort_by` | enum | Sort field | `?sort_by=name` (name, level, created_at, updated_at) |
| `sort_direction` | enum | Sort order | `?sort_direction=asc` (asc, desc) |
| `page` | integer | Page number | `?page=2` |
| `per_page` | integer | Items per page | `?per_page=24` (min: 1, max: 100) |
| `search` | string | Search (max 255 chars) | `?search=fire` |

### Meilisearch `filter` Parameter

**Operators:** `=`, `!=`, `>`, `>=`, `<`, `<=`, `IN`, `IS EMPTY`, `AND`, `OR`

**Available Filterable Fields:**

| Field | Type | Values | Example Filter |
|-------|------|--------|----------------|
| `level` | int | 0-9 | `level = 3` |
| `school_code` | string | AB, CN, DV, EN, EV, IL, NE, TR | `school_code = EV` |
| `school_name` | string | "Abjuration", "Conjuration", etc. | `school_name = "Evocation"` |
| `concentration` | bool | true, false | `concentration = true` |
| `ritual` | bool | true, false | `ritual = false` |
| `class_slugs` | array | wizard, bard, cleric, etc. | `class_slugs IN [wizard, bard]` |
| `tag_slugs` | array | damage, healing, buff, etc. | `tag_slugs IN [damage, healing]` |
| `source_codes` | array | PHB, XGE, TCE, etc. | `source_codes IN [PHB]` |
| `damage_types` | array | F, C, O, A, T, Li, N, P, PS, R, Fo, Ne | `damage_types IN [F, C]` |
| `saving_throws` | array | STR, DEX, CON, INT, WIS, CHA | `saving_throws IN [DEX, WIS]` |
| `requires_verbal` | bool | true, false | `requires_verbal = true` |
| `requires_somatic` | bool | true, false | `requires_somatic = false` |
| `requires_material` | bool | true, false | `requires_material = true` |

**Notable Absences (NOT in Meilisearch):**
- ❌ `casting_time` - NOT filterable in Meilisearch
- ❌ `range` - NOT filterable in Meilisearch
- ❌ `duration` - NOT filterable in Meilisearch (except concentration flag)
- ❌ `has_higher_levels` - NOT filterable in Meilisearch

---

## 🔍 Current Frontend Implementation Analysis

### Current Filters (spells/index.vue)

| Filter | Current Implementation | API Support | Status |
|--------|----------------------|-------------|--------|
| **Level** | `params.level = selectedLevel.value` (MySQL) | ✅ `filter=level = 3` | 🔴 BROKEN - Using deprecated param |
| **School** | `params.school = selectedSchool.value` (MySQL) | ✅ `filter=school_code = EV` | 🔴 BROKEN - Using deprecated param |
| **Class** | `filter=class_slugs IN [wizard]` (Meilisearch) | ✅ `filter=class_slugs IN [wizard]` | ✅ WORKING |
| **Concentration** | `params.concentration = concentrationFilter.value` (MySQL) | ✅ `filter=concentration = true` | 🔴 BROKEN - Using deprecated param |
| **Ritual** | `params.ritual = ritualFilter.value` (MySQL) | ✅ `filter=ritual = true` | 🔴 BROKEN - Using deprecated param |
| **Damage Types** | `params.damage_type = "F,C"` (CSV) | ✅ `filter=damage_types IN [F, C]` | 🔴 BROKEN - Using deprecated param |
| **Saving Throws** | `params.saving_throw = "DEX,WIS"` (CSV) | ✅ `filter=saving_throws IN [DEX, WIS]` | 🔴 BROKEN - Using deprecated param |
| **Verbal Component** | `params.has_verbal = "1"` (MySQL) | ✅ `filter=requires_verbal = true` | 🔴 BROKEN - Using deprecated param |
| **Somatic Component** | `params.has_somatic = "1"` (MySQL) | ✅ `filter=requires_somatic = true` | 🔴 BROKEN - Using deprecated param |
| **Material Component** | `params.has_material = "1"` (MySQL) | ✅ `filter=requires_material = true` | 🔴 BROKEN - Using deprecated param |
| **Higher Levels** | `params.has_higher_levels = "1"` (MySQL) | ❌ NOT AVAILABLE | 🔴 IMPOSSIBLE - Remove filter |
| **Casting Time** | `params.casting_time = "1 action"` (MySQL) | ❌ NOT AVAILABLE | 🔴 IMPOSSIBLE - Remove filter |
| **Range** | `params.range = "60 feet"` (MySQL) | ❌ NOT AVAILABLE | 🔴 IMPOSSIBLE - Remove filter |
| **Duration** | `params.duration = "Instantaneous"` (MySQL) | ❌ NOT AVAILABLE | 🔴 IMPOSSIBLE - Remove filter |

### Summary Statistics

- **Total Filters:** 14
- **Working Correctly:** 1 (class)
- **Broken (Using Deprecated Params):** 9
- **Impossible (Field Not Filterable):** 4
- **Success Rate:** 7% 🔴

---

## 🎯 Migration Plan

### Phase 1: Remove Impossible Filters ❌

**Remove these filters entirely (not supported by Meilisearch):**

1. ❌ `has_higher_levels` filter → Remove UI component
2. ❌ `casting_time` filter → Remove UI component
3. ❌ `range` filter → Remove UI component
4. ❌ `duration` filter → Remove UI component

**Rationale:**
- These fields are NOT indexed in Meilisearch
- Cannot filter on arbitrary text fields (performance/complexity)
- Concentration flag covers most duration use cases
- Users can still search for these values with full-text search (`q` parameter)

---

### Phase 2: Migrate Working Filters to Meilisearch ✅

**Convert these filters from deprecated MySQL params to Meilisearch syntax:**

#### 1. **Level Filter**
```typescript
// Before (BROKEN)
if (selectedLevel.value !== null) params.level = selectedLevel.value

// After (WORKING)
if (selectedLevel.value !== null) {
  meilisearchFilters.push(`level = ${selectedLevel.value}`)
}
```

#### 2. **School Filter**
```typescript
// Before (BROKEN)
if (selectedSchool.value !== null) params.school = selectedSchool.value

// After (WORKING)
// NOTE: Need to map ID to school_code or school_name
const selectedSchoolCode = spellSchools.value?.find(s => s.id === selectedSchool.value)?.code
if (selectedSchoolCode) {
  meilisearchFilters.push(`school_code = ${selectedSchoolCode}`)
}
```

**⚠️ CRITICAL:** School filter currently uses `id` (number), but API expects `school_code` (string) or `school_name` (string).
**Action Required:** Update UI to use school codes (AB, CN, DV, etc.) instead of IDs.

#### 3. **Concentration Filter**
```typescript
// Before (BROKEN)
if (concentrationFilter.value !== null) params.concentration = concentrationFilter.value

// After (WORKING)
if (concentrationFilter.value !== null) {
  const boolValue = concentrationFilter.value === '1' || concentrationFilter.value === 'true'
  meilisearchFilters.push(`concentration = ${boolValue}`)
}
```

#### 4. **Ritual Filter**
```typescript
// Before (BROKEN)
if (ritualFilter.value !== null) params.ritual = ritualFilter.value

// After (WORKING)
if (ritualFilter.value !== null) {
  const boolValue = ritualFilter.value === '1' || ritualFilter.value === 'true'
  meilisearchFilters.push(`ritual = ${boolValue}`)
}
```

#### 5. **Damage Types Filter** (Multi-select)
```typescript
// Before (BROKEN)
if (selectedDamageTypes.value.length > 0) {
  params.damage_type = selectedDamageTypes.value.join(',')
}

// After (WORKING)
if (selectedDamageTypes.value.length > 0) {
  const codes = selectedDamageTypes.value.join(', ')
  meilisearchFilters.push(`damage_types IN [${codes}]`)
}
```

#### 6. **Saving Throws Filter** (Multi-select)
```typescript
// Before (BROKEN)
if (selectedSavingThrows.value.length > 0) {
  params.saving_throw = selectedSavingThrows.value.join(',')
}

// After (WORKING)
if (selectedSavingThrows.value.length > 0) {
  const codes = selectedSavingThrows.value.join(', ')
  meilisearchFilters.push(`saving_throws IN [${codes}]`)
}
```

#### 7. **Verbal Component Filter**
```typescript
// Before (BROKEN)
if (verbalFilter.value !== null) params.has_verbal = verbalFilter.value

// After (WORKING)
if (verbalFilter.value !== null) {
  const boolValue = verbalFilter.value === '1' || verbalFilter.value === 'true'
  meilisearchFilters.push(`requires_verbal = ${boolValue}`)
}
```

#### 8. **Somatic Component Filter**
```typescript
// Before (BROKEN)
if (somaticFilter.value !== null) params.has_somatic = somaticFilter.value

// After (WORKING)
if (somaticFilter.value !== null) {
  const boolValue = somaticFilter.value === '1' || somaticFilter.value === 'true'
  meilisearchFilters.push(`requires_somatic = ${boolValue}`)
}
```

#### 9. **Material Component Filter**
```typescript
// Before (BROKEN)
if (materialFilter.value !== null) params.has_material = materialFilter.value

// After (WORKING)
if (materialFilter.value !== null) {
  const boolValue = materialFilter.value === '1' || materialFilter.value === 'true'
  meilisearchFilters.push(`requires_material = ${boolValue}`)
}
```

---

### Phase 3: Clean Up Query Builder

**After migration, the query builder should ONLY use Meilisearch filters:**

```typescript
const queryBuilder = computed(() => {
  const params: Record<string, unknown> = {}
  const meilisearchFilters: string[] = []

  // Level filter
  if (selectedLevel.value !== null) {
    meilisearchFilters.push(`level = ${selectedLevel.value}`)
  }

  // School filter (need school code from ID)
  if (selectedSchool.value !== null) {
    const schoolCode = spellSchools.value?.find(s => s.id === selectedSchool.value)?.code
    if (schoolCode) {
      meilisearchFilters.push(`school_code = ${schoolCode}`)
    }
  }

  // Class filter
  if (selectedClass.value !== null) {
    meilisearchFilters.push(`class_slugs IN [${selectedClass.value}]`)
  }

  // Concentration filter
  if (concentrationFilter.value !== null) {
    const boolValue = concentrationFilter.value === '1' || concentrationFilter.value === 'true'
    meilisearchFilters.push(`concentration = ${boolValue}`)
  }

  // Ritual filter
  if (ritualFilter.value !== null) {
    const boolValue = ritualFilter.value === '1' || ritualFilter.value === 'true'
    meilisearchFilters.push(`ritual = ${boolValue}`)
  }

  // Damage types (multi-select)
  if (selectedDamageTypes.value.length > 0) {
    const codes = selectedDamageTypes.value.join(', ')
    meilisearchFilters.push(`damage_types IN [${codes}]`)
  }

  // Saving throws (multi-select)
  if (selectedSavingThrows.value.length > 0) {
    const codes = selectedSavingThrows.value.join(', ')
    meilisearchFilters.push(`saving_throws IN [${codes}]`)
  }

  // Component filters
  if (verbalFilter.value !== null) {
    const boolValue = verbalFilter.value === '1' || verbalFilter.value === 'true'
    meilisearchFilters.push(`requires_verbal = ${boolValue}`)
  }

  if (somaticFilter.value !== null) {
    const boolValue = somaticFilter.value === '1' || somaticFilter.value === 'true'
    meilisearchFilters.push(`requires_somatic = ${boolValue}`)
  }

  if (materialFilter.value !== null) {
    const boolValue = materialFilter.value === '1' || materialFilter.value === 'true'
    meilisearchFilters.push(`requires_material = ${boolValue}`)
  }

  // Combine all filters
  if (meilisearchFilters.length > 0) {
    params.filter = meilisearchFilters.join(' AND ')
  }

  return params
})
```

---

### Phase 4: UI Cleanup

**Remove filter UI components for unsupported filters:**

1. Remove `higherLevelsFilter` ref and UI component
2. Remove `castingTimeFilter` ref and UI component
3. Remove `rangeFilter` ref and UI component
4. Remove `durationFilter` ref and UI component

**Update filter count calculation:**
- Remove counts for deleted filters

**Update `clearFilters()` function:**
- Remove references to deleted filters

---

## 🧪 Testing Plan

### Test Each Filter Individually

```bash
# Level filter
curl "http://localhost:8080/api/v1/spells?filter=level%20%3D%203&per_page=5" | jq '.meta.total'

# School filter (Evocation)
curl "http://localhost:8080/api/v1/spells?filter=school_code%20%3D%20EV&per_page=5" | jq '.meta.total'

# Class filter
curl "http://localhost:8080/api/v1/spells?filter=class_slugs%20IN%20%5Bwizard%5D&per_page=5" | jq '.meta.total'

# Concentration filter
curl "http://localhost:8080/api/v1/spells?filter=concentration%20%3D%20true&per_page=5" | jq '.meta.total'

# Ritual filter
curl "http://localhost:8080/api/v1/spells?filter=ritual%20%3D%20true&per_page=5" | jq '.meta.total'

# Damage types (Fire OR Cold)
curl "http://localhost:8080/api/v1/spells?filter=damage_types%20IN%20%5BF%2C%20C%5D&per_page=5" | jq '.meta.total'

# Saving throws (DEX OR WIS)
curl "http://localhost:8080/api/v1/spells?filter=saving_throws%20IN%20%5BDEX%2C%20WIS%5D&per_page=5" | jq '.meta.total'

# Verbal component
curl "http://localhost:8080/api/v1/spells?filter=requires_verbal%20%3D%20true&per_page=5" | jq '.meta.total'

# Combined filters (3rd level wizard evocation spells)
curl "http://localhost:8080/api/v1/spells?filter=level%20%3D%203%20AND%20class_slugs%20IN%20%5Bwizard%5D%20AND%20school_code%20%3D%20EV&per_page=5" | jq '.meta.total'
```

### Frontend Testing

```bash
# Test each filter in browser
for filter in level school class concentration ritual damage_type saving_throw verbal somatic material; do
  echo "Testing $filter filter..."
  curl -s "http://localhost:3000/spells?$filter=..." -o /dev/null -w "HTTP %{http_code}\n"
done
```

---

## 📊 Expected Results

### Filters Working After Migration
- ✅ Level (10 options: Cantrip, 1-9)
- ✅ School (8 options: Abjuration, Conjuration, Divination, Enchantment, Evocation, Illusion, Necromancy, Transmutation)
- ✅ Class (15 base classes)
- ✅ Concentration (Yes/No)
- ✅ Ritual (Yes/No)
- ✅ Damage Types (Multi-select: Fire, Cold, Acid, etc.)
- ✅ Saving Throws (Multi-select: STR, DEX, CON, INT, WIS, CHA)
- ✅ Verbal Component (Yes/No)
- ✅ Somatic Component (Yes/No)
- ✅ Material Component (Yes/No)

**Total: 10 working filters** (down from 14)

### Filters Removed (Not Supported)
- ❌ Has Higher Levels
- ❌ Casting Time
- ❌ Range
- ❌ Duration

**Rationale:** These fields are not indexed in Meilisearch. Users can search for text values using full-text search (`?q=1 action`).

---

## 🚀 New Capabilities Unlocked

### Advanced Multi-Filter Queries

**High-level wizard evocation spells:**
```
filter=level >= 7 AND class_slugs IN [wizard] AND school_code = EV
```

**Low-level cleric healing spells (non-concentration):**
```
filter=level <= 3 AND class_slugs IN [cleric] AND tag_slugs IN [healing] AND concentration = false
```

**Ritual spells for bard or wizard:**
```
filter=ritual = true AND class_slugs IN [bard, wizard]
```

**Fire or cold damage spells requiring DEX save:**
```
filter=damage_types IN [F, C] AND saving_throws IN [DEX]
```

**Spells with no verbal component (silent casting):**
```
filter=requires_verbal = false
```

---

## 💡 Key Decisions

### 1. **Remove Unsupported Filters**
**Decision:** Remove casting_time, range, duration, has_higher_levels filters
**Rationale:** Not indexed in Meilisearch, cannot be filtered efficiently
**Alternative:** Users can search text with `?q=1 action` or `?q=60 feet`

### 2. **School Filter: Use Code Instead of ID**
**Decision:** Change school filter to use `school_code` (AB, CN, DV, etc.) instead of ID
**Rationale:** Meilisearch uses `school_code`, not numeric IDs
**Action Required:** Update `selectedSchool` ref to store string code instead of number ID

### 3. **Boolean Filters: String to Boolean Conversion**
**Decision:** Convert string values ('1', 'true') to boolean (true, false) for Meilisearch
**Rationale:** Meilisearch expects boolean values, not strings
**Implementation:** `concentration = true` not `concentration = "1"`

### 4. **Multi-Select Filters: CSV to IN Syntax**
**Decision:** Convert comma-separated values to `IN [...]` syntax
**Before:** `damage_type=F,C`
**After:** `filter=damage_types IN [F, C]`

---

## 📁 Files to Modify

### 1. `app/pages/spells/index.vue`
**Changes:**
- Remove `higherLevelsFilter`, `castingTimeFilter`, `rangeFilter`, `durationFilter` refs
- Update `selectedSchool` to use string code instead of number ID
- Rewrite `queryBuilder` to use only Meilisearch filters
- Remove deleted filters from `clearFilters()`
- Update `activeFilterCount` calculation
- Remove UI components for deleted filters

### 2. `app/types/api/entities.ts` (if needed)
**Changes:**
- Verify `SpellSchool` type has `code` property
- Add comments about Meilisearch filterable fields

### 3. `CHANGELOG.md`
**Changes:**
- Document breaking change: "Migrated spell filters to Meilisearch-only syntax"
- List removed filters
- List updated filters

---

## 🎯 Success Criteria

- [ ] All 10 supported filters working with Meilisearch syntax
- [ ] Zero MySQL fallback params in query builder
- [ ] 4 unsupported filters removed from UI
- [ ] TypeScript compilation clean
- [ ] All filter tests passing (API + Frontend)
- [ ] Combined filters working (e.g., level + class + school)
- [ ] CHANGELOG.md updated
- [ ] Changes committed

---

## 📚 Related Documentation

- **Previous Migration:** `docs/HANDOVER-2025-11-25-MEILISEARCH-API-MIGRATION.md` (class filter only)
- **API Spec:** `http://localhost:8080/docs/api.json`
- **Meilisearch Docs:** Filter syntax reference

---

## 👤 Next Developer Notes

**This is a BREAKING CHANGE.**

The API no longer supports individual MySQL params like `?level=3&school=2&concentration=1`. Everything must go through the `filter` parameter using Meilisearch syntax.

**Before implementing:**
1. Read this entire document
2. Test API directly with curl to verify filter syntax
3. Update one filter at a time
4. Test each filter individually before moving to next
5. Commit after each working filter (not all at once)

**After implementing:**
- Frontend will have 10 working filters (down from 14)
- Users can still search for removed filter values using full-text search (`?q=1 action`)
- Advanced multi-filter queries will be possible
- Performance will improve (Meilisearch is faster than MySQL)

---

**End of Audit**
