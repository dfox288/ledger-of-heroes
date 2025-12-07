# API Filtering & Search Enhancement Analysis

**Date:** 2025-11-23
**Purpose:** Comprehensive analysis of API filtering capabilities vs current frontend implementation
**Status:** 🔍 Analysis Complete - Ready for Implementation Planning

---

## Executive Summary

The backend API exposes **90+ query parameters** across 7 entity endpoints with rich filtering capabilities. The frontend currently uses **only ~30%** of these features. This analysis identifies **60+ unused filtering opportunities** that can significantly enhance user experience.

### Key Findings

| Entity | API Filters | Frontend Filters | Utilization | Opportunity |
|--------|------------|------------------|-------------|-------------|
| **Spells** | 13 filters | 2 filters (level, school) | 15% | 🔴 **HIGH** |
| **Items** | 12 filters | 3 filters (type, rarity, magic) | 25% | 🔴 **HIGH** |
| **Monsters** | 14 filters | 2 filters (CR, type) | 14% | 🔴 **HIGH** |
| **Races** | 18 filters | 1 filter (size) | 6% | 🔴 **CRITICAL** |
| **Classes** | 14 filters | 0 filters | 0% | 🔴 **CRITICAL** |
| **Backgrounds** | 9 filters | 0 filters | 0% | 🟡 **MEDIUM** |
| **Feats** | 11 filters | 0 filters | 0% | 🟡 **MEDIUM** |

**Overall Utilization: 13% (13 of 90 filters implemented)**

---

## 1. Spells (13 API Filters Available)

### Currently Implemented ✅
1. `level` - Spell level filter (0-9) ✅
2. `school` - School of magic filter ✅
3. `q` - Search query ✅

### Available But Unused (10 filters) 🚀

#### 🔴 High-Impact Filters
- **`concentration`** (boolean) - Filter ritual-only or non-ritual spells
  - *Use Case:* "Show me only concentration spells" / "No concentration spells"
  - *UI:* Toggle or tri-state button (All / Concentration / Non-Concentration)

- **`ritual`** (boolean) - Filter ritual-capable spells
  - *Use Case:* "Show only ritual spells for my wizard"
  - *UI:* Toggle or tri-state button

- **`damage_type`** (string) - Filter by damage type (fire, cold, necrotic, etc.)
  - *Use Case:* "Show fire damage spells" / "Cold damage for my white dragon sorcerer"
  - *UI:* Multi-select dropdown with damage type icons/colors

- **`saving_throw`** (string) - Filter by saving throw type (DEX, CON, WIS, etc.)
  - *Use Case:* "Show spells that target DEX saves" / "Find spells bypassing high CON"
  - *UI:* Multi-select dropdown with ability score abbreviations

#### 🟡 Medium-Impact Filters
- **`requires_verbal`** (boolean) - Filter by verbal component requirement
  - *Use Case:* "No verbal components (silenced condition)"
  - *UI:* Component filter group (V/S/M toggles)

- **`requires_somatic`** (boolean) - Filter by somatic component requirement
  - *Use Case:* "No somatic components (restrained)"
  - *UI:* Component filter group

- **`requires_material`** (boolean) - Filter by material component requirement
  - *Use Case:* "No material components (no component pouch)"
  - *UI:* Component filter group

#### 🟢 Low-Impact Filters
- **`sort_by`** (name, level, created_at, updated_at) - Sorting options
  - *Use Case:* "Sort by name" / "Sort by level (high to low)"
  - *UI:* Sort dropdown in header

- **`sort_direction`** (asc, desc) - Sort direction
  - *UI:* Toggle with sort_by

- **`filter`** (Meilisearch expression) - Advanced power-user filtering
  - *Use Case:* Complex queries like `level > 3 AND school_code = "EV" AND concentration = false`
  - *UI:* Advanced filter modal or query builder (future enhancement)

---

## 2. Items (12 API Filters Available)

### Currently Implemented ✅
1. `type` - Item type filter ✅
2. `rarity` - Rarity filter ✅
3. `is_magic` - Magic/non-magic filter (as "magic filter") ✅
4. `q` - Search query ✅

### Available But Unused (8 filters) 🚀

#### 🔴 High-Impact Filters
- **`has_charges`** (boolean) - Filter items with charge mechanics
  - *Use Case:* "Show only items with charges" / "Find wands and staffs"
  - *UI:* Toggle or tri-state button

- **`has_prerequisites`** (boolean) - Filter items requiring attunement prerequisites
  - *Use Case:* "Items I can attune to now" vs "Items with class/race restrictions"
  - *UI:* Toggle or tri-state button

#### 🟡 Medium-Impact Filters
- **`spells`** (comma-separated slugs) - Filter items that grant specific spells
  - *Use Case:* "Find items that let me cast Fireball" / "Healing spell items"
  - *UI:* Spell search/autocomplete dropdown
  - *Note:* Requires spell reference data

- **`spells_operator`** (AND/OR) - Spell filter logic
  - *Use Case:* "Items with Fireball AND Lightning Bolt" vs "Either spell"
  - *UI:* Toggle when multiple spells selected

- **`spell_level`** (0-9) - Filter items by granted spell level
  - *Use Case:* "Items granting 3rd level or higher spells"
  - *UI:* Dropdown or slider

- **`min_strength`** (integer) - Minimum STR requirement for armor
  - *Use Case:* "Heavy armor my character can wear (STR 15+)"
  - *UI:* Number input or slider

#### 🟢 Low-Impact Filters
- **`sort_by`** (name, type, rarity, created_at, updated_at)
- **`sort_direction`** (asc, desc)
- **`filter`** (Meilisearch expression) - Advanced filtering

---

## 3. Monsters (14 API Filters Available)

### Currently Implemented ✅
1. `challenge_rating` - Exact CR filter ✅
2. `type` - Monster type filter ✅
3. `q` - Search query ✅

### Available But Unused (11 filters) 🚀

#### 🔴 High-Impact Filters
- **`min_cr`** + **`max_cr`** (number) - CR range filtering
  - *Use Case:* "CR 3-7 for my level 5 party" / "Boss monsters CR 15+"
  - *UI:* Dual slider or two dropdowns for range
  - *Note:* More flexible than exact `challenge_rating`

- **`size`** (T, S, M, L, H, G) - Size category filter
  - *Use Case:* "Large or bigger monsters" / "Small creatures for my dungeon"
  - *UI:* Multi-select dropdown with size names

- **`alignment`** (string) - Alignment filter
  - *Use Case:* "Lawful Evil enemies" / "Chaotic monsters"
  - *UI:* Multi-select dropdown with alignment grid

#### 🟡 Medium-Impact Filters
- **`spells`** (comma-separated slugs) - Monsters that can cast specific spells
  - *Use Case:* "Find spellcasters with Counterspell" / "Monsters using fire magic"
  - *UI:* Spell search/autocomplete

- **`spells_operator`** (AND/OR) - Spell filter logic
- **`spell_level`** (0-9) - Monsters casting spells of certain level
  - *Use Case:* "Spellcasters with 9th level spells" / "Cantrip-only casters"
  - *UI:* Dropdown

- **`spellcasting_ability`** (INT, WIS, CHA) - Filter by spellcasting ability
  - *Use Case:* "Wizard-type monsters (INT)" / "Cleric-like (WIS)"
  - *UI:* Multi-select dropdown

#### 🟢 Low-Impact Filters
- **`sort_by`** (name, challenge_rating, hit_points_average, armor_class, experience_points)
  - *Use Case:* "Sort by highest CR" / "Sort by most HP"
  - *UI:* Sort dropdown

- **`sort_direction`** (asc, desc)
- **`filter`** (Meilisearch expression) - Advanced filtering

---

## 4. Races (18 API Filters Available - MOST UNDERUTILIZED)

### Currently Implemented ✅
1. `size` - Size filter ✅ (but noted as **not working** in backend - see CLAUDE.md line 410-413)
2. `q` - Search query ✅

### Available But Unused (16 filters!) 🚀

#### 🔴 High-Impact Filters
- **`ability_bonus`** (STR, DEX, CON, INT, WIS, CHA) - Filter by ability score bonuses
  - *Use Case:* "Races with DEX bonus for my rogue" / "+INT races for wizard"
  - *UI:* Multi-select dropdown with ability score chips

- **`min_speed`** (integer) - Minimum walking speed
  - *Use Case:* "Fast races (35+ ft speed)" / "Slow but sturdy races"
  - *UI:* Slider or dropdown (25, 30, 35, 40+ ft)

- **`has_darkvision`** (boolean) - Darkvision filter
  - *Use Case:* "Races with darkvision for Underdark campaign"
  - *UI:* Toggle or tri-state button

- **`speaks_language`** (string) - Filter by language proficiency
  - *Use Case:* "Races speaking Draconic" / "Common + Elvish"
  - *UI:* Multi-select dropdown with language names

#### 🟡 Medium-Impact Filters
- **`grants_proficiency`** (string) - Filter by granted proficiency
  - *Use Case:* "Races with weapon proficiencies" / "Armor proficiencies"
  - *UI:* Proficiency search/autocomplete

- **`grants_skill`** (string) - Filter by granted skill proficiencies
  - *Use Case:* "Races with Perception proficiency" / "Stealth bonus"
  - *UI:* Multi-select dropdown with skills

- **`grants_proficiency_type`** (string) - Filter by proficiency category
  - *Use Case:* "Races granting martial weapons" / "Artisan's tools"
  - *UI:* Multi-select dropdown

- **`language_choice_count`** (integer) - Filter by number of language choices granted
  - *Use Case:* "Races with extra language choice" / "No language choices"
  - *UI:* Dropdown (0, 1, 2, 3+ choices)

- **`grants_languages`** (boolean) - Filter races that grant ANY languages
  - *Use Case:* "Races with automatic language grants"
  - *UI:* Toggle

- **`has_innate_spells`** (boolean) - Races with innate spellcasting
  - *Use Case:* "Races with racial spells" / "Tiefling-like magic"
  - *UI:* Toggle or tri-state button

#### 🟡 Medium-Impact (Spell Filters)
- **`spells`** (comma-separated slugs) - Filter by racial spells
  - *Use Case:* "Races that can cast Disguise Self"
  - *UI:* Spell search

- **`spells_operator`** (AND/OR) - Spell filter logic
- **`spell_level`** (0-9) - Filter by racial spell level

#### 🟢 Low-Impact Filters
- **`sort_by`** (name, size, speed, created_at, updated_at)
- **`sort_direction`** (asc, desc)
- **`filter`** (Meilisearch expression)

---

## 5. Classes (14 API Filters Available - 0% UTILIZED)

### Currently Implemented ✅
1. `q` - Search query ✅

### Available But Unused (13 filters!) 🚀

#### 🔴 High-Impact Filters
- **`base_only`** (boolean) - Filter base classes vs subclasses
  - *Use Case:* "Show only base classes (Fighter, Wizard, etc.)" / "Show subclasses too"
  - *UI:* Toggle or filter chip

- **`is_spellcaster`** (boolean) - Spellcasting capability filter
  - *Use Case:* "Show only spellcasting classes" / "Non-spellcasters for martial build"
  - *UI:* Toggle or tri-state button

- **`hit_die`** (6, 8, 10, 12) - Hit die filter
  - *Use Case:* "d10 or d12 hit die classes (high HP)" / "d6 hit die (glass cannons)"
  - *UI:* Multi-select dropdown with hit die options

- **`max_spell_level`** (1-9) - Maximum spell level the class can cast
  - *Use Case:* "Full casters (9th level spells)" / "Half casters (5th level max)"
  - *UI:* Dropdown or slider

#### 🟡 Medium-Impact Filters
- **`grants_proficiency`** (string) - Filter by granted proficiency
  - *Use Case:* "Classes with heavy armor proficiency" / "Martial weapon classes"
  - *UI:* Proficiency search/autocomplete

- **`grants_skill`** (string) - Filter by skill proficiency options
  - *Use Case:* "Classes with Stealth proficiency" / "Perception classes"
  - *UI:* Multi-select dropdown with skills

- **`grants_saving_throw`** (STR, DEX, CON, INT, WIS, CHA) - Saving throw proficiencies
  - *Use Case:* "Classes with DEX save proficiency" / "WIS save classes"
  - *UI:* Multi-select dropdown with ability scores

#### 🟡 Medium-Impact (Spell Filters)
- **`spells`** (comma-separated slugs) - Classes with specific spells on spell list
  - *Use Case:* "Classes that can learn Fireball" / "Healing spell classes"
  - *UI:* Spell search

- **`spells_operator`** (AND/OR)
- **`spell_level`** (0-9) - Classes with spells of certain level

#### 🟢 Low-Impact Filters
- **`sort_by`** (name, hit_die, created_at, updated_at)
- **`sort_direction`** (asc, desc)
- **`filter`** (Meilisearch expression)

---

## 6. Backgrounds (9 API Filters Available - 0% UTILIZED)

### Currently Implemented ✅
1. `q` - Search query ✅

### Available But Unused (8 filters) 🚀

#### 🟡 Medium-Impact Filters
- **`grants_proficiency`** (string) - Filter by granted proficiency
  - *Use Case:* "Backgrounds with vehicle proficiencies" / "Gaming set backgrounds"
  - *UI:* Proficiency search/autocomplete

- **`grants_skill`** (string) - Filter by skill proficiency grants
  - *Use Case:* "Backgrounds with Persuasion" / "Stealth backgrounds"
  - *UI:* Multi-select dropdown with skills

- **`speaks_language`** (string) - Filter by language grants
  - *Use Case:* "Backgrounds granting Draconic" / "Exotic language backgrounds"
  - *UI:* Multi-select dropdown with languages

- **`language_choice_count`** (integer) - Filter by number of language choices
  - *Use Case:* "Backgrounds with 2+ language choices"
  - *UI:* Dropdown

- **`grants_languages`** (boolean) - Filter backgrounds granting ANY languages
  - *Use Case:* "Backgrounds with automatic language grants"
  - *UI:* Toggle

#### 🟢 Low-Impact Filters
- **`sort_by`** (name, created_at, updated_at)
- **`sort_direction`** (asc, desc)
- **`filter`** (Meilisearch expression)

---

## 7. Feats (11 API Filters Available - 0% UTILIZED)

### Currently Implemented ✅
1. `q` - Search query ✅

### Available But Unused (10 filters) 🚀

#### 🔴 High-Impact Filters
- **`has_prerequisites`** (boolean) - Filter feats with/without prerequisites
  - *Use Case:* "Feats I can take now (no prerequisites)" / "Advanced feats (with prereqs)"
  - *UI:* Toggle or tri-state button

- **`prerequisite_ability`** (STR, DEX, CON, INT, WIS, CHA) - Filter by ability score prerequisite
  - *Use Case:* "Feats requiring DEX 13+" / "STR-based feats"
  - *UI:* Multi-select dropdown with ability scores

- **`min_value`** (integer) - Minimum ability score value for prerequisite
  - *Use Case:* "Feats requiring 15+ in any ability" / "13+ feats only"
  - *UI:* Dropdown (13, 14, 15, 16, etc.)

#### 🟡 Medium-Impact Filters
- **`prerequisite_race`** (string) - Filter by race prerequisite
  - *Use Case:* "Dwarf-only feats" / "Elf feats"
  - *UI:* Multi-select dropdown with races

- **`prerequisite_proficiency`** (string) - Filter by proficiency prerequisite
  - *Use Case:* "Feats requiring martial weapon proficiency" / "Heavy armor feats"
  - *UI:* Proficiency search

- **`grants_proficiency`** (string) - Filter by granted proficiency
  - *Use Case:* "Feats granting weapon proficiencies" / "Armor proficiency feats"
  - *UI:* Proficiency search/autocomplete

- **`grants_skill`** (string) - Filter by granted skill proficiency
  - *Use Case:* "Feats with skill proficiencies" / "Stealth feat"
  - *UI:* Multi-select dropdown with skills

#### 🟢 Low-Impact Filters
- **`sort_by`** (name, created_at, updated_at)
- **`sort_direction`** (asc, desc)
- **`filter`** (Meilisearch expression)

---

## Summary by Priority

### 🔴 High-Priority Enhancements (Immediate Value)

#### Spells (4 new filters)
1. **Concentration filter** - Toggle (All / Concentration / Non-Concentration)
2. **Ritual filter** - Toggle (All / Ritual / Non-Ritual)
3. **Damage type filter** - Multi-select dropdown
4. **Saving throw filter** - Multi-select dropdown

#### Items (2 new filters)
1. **Has charges filter** - Toggle
2. **Has prerequisites filter** - Toggle

#### Monsters (3 new filters)
1. **CR range filter** (min_cr + max_cr) - Dual slider or range dropdowns
2. **Size filter** - Multi-select dropdown
3. **Alignment filter** - Multi-select dropdown or alignment grid

#### Races (4 new filters)
1. **Ability bonus filter** - Multi-select dropdown (STR, DEX, CON, INT, WIS, CHA)
2. **Darkvision filter** - Toggle
3. **Language filter** - Multi-select dropdown
4. **Min speed filter** - Slider or dropdown

#### Classes (4 new filters)
1. **Base vs subclass toggle** - Toggle
2. **Spellcaster filter** - Toggle (All / Spellcasters / Non-Spellcasters)
3. **Hit die filter** - Multi-select dropdown (d6, d8, d10, d12)
4. **Max spell level filter** - Dropdown (full caster / half caster / third caster)

#### Feats (3 new filters)
1. **Has prerequisites toggle** - Toggle
2. **Prerequisite ability filter** - Multi-select dropdown
3. **Min ability value filter** - Dropdown

**Total High-Priority Filters: 20 new filters**

---

### 🟡 Medium-Priority Enhancements (Nice to Have)

- Component filters for spells (V/S/M toggles)
- Spell-granting filters (items, races, classes)
- Proficiency filters (races, classes, backgrounds, feats)
- Skill filters (all entities)
- Language filters (races, backgrounds)

**Total Medium-Priority Filters: ~25 new filters**

---

### 🟢 Low-Priority Enhancements (Power Users)

- Sort controls on all entity pages (sort_by + sort_direction)
- Advanced Meilisearch filter expressions (query builder)

**Total Low-Priority Filters: ~15 new filters**

---

## Implementation Recommendations

### Phase 1: High-Impact Quick Wins (Est: 2-3 days)
- **Spells:** Add concentration + ritual toggles (simple boolean filters)
- **Items:** Add has_charges + has_prerequisites toggles
- **Monsters:** Add size multi-select (reuse Size entity data)
- **Feats:** Add has_prerequisites toggle
- **Classes:** Add base_only toggle

**Effort:** Low (reuse existing filter UI patterns)
**Impact:** High (5 entities improved with 8 new filters)

### Phase 2: Complex Filters (Est: 3-4 days)
- **Spells:** Damage type + saving throw multi-selects
- **Monsters:** CR range slider + alignment grid
- **Races:** Ability bonus + language multi-selects
- **Classes:** Hit die + spellcaster filters

**Effort:** Medium (new UI components needed)
**Impact:** High (4 entities with 8 major filters)

### Phase 3: Advanced Filters (Est: 5-7 days)
- Spell-granting filters (items, races, classes)
- Proficiency filters (all entities)
- Component filters for spells
- Skill filters (all entities)

**Effort:** High (requires reference data lookups, complex UI)
**Impact:** Medium-High (power user features)

### Phase 4: Sorting & Polish (Est: 2-3 days)
- Add sort controls to all entity pages
- Implement sort persistence in URL params
- Add sort indicators to filter bar

**Effort:** Low (consistent pattern across all pages)
**Impact:** Medium (nice QOL improvement)

---

## Technical Considerations

### UI/UX Patterns
1. **Filter Bar Layout** - Group related filters (basic → advanced)
2. **Active Filter Chips** - Show all active filters with clear buttons
3. **Filter Persistence** - Save filters in URL query params
4. **Filter State** - Update route on filter change, restore from URL on page load
5. **Mobile Responsiveness** - Collapsible filter panel on mobile

### Components to Create
1. **`<UiFilterToggle>`** - Tri-state toggle (All / Yes / No)
2. **`<UiFilterMultiSelect>`** - Multi-select dropdown with chips
3. **`<UiFilterRangeSlider>`** - Dual-handle slider for min/max
4. **`<UiFilterAlignmentGrid>`** - 3x3 grid for D&D alignment selection
5. **`<UiFilterSortControl>`** - Sort field + direction dropdown combo

### Testing Strategy
1. **Unit Tests** - Test queryBuilder logic for each filter combination
2. **Component Tests** - Test filter UI components in isolation
3. **Integration Tests** - Test filter → API → results flow
4. **E2E Tests** - Test user workflows (apply filters, clear, bookmark)

---

## ROI Analysis

### Current State
- **13 filters** across 7 entities
- **Utilization:** 13% of available API capabilities
- **User Experience:** Basic filtering only

### After Phase 1 (High-Impact Quick Wins)
- **21 filters** (+8 new filters, +62% increase)
- **Utilization:** 23% of available API capabilities
- **Estimated Implementation:** 2-3 days
- **User Experience:** Major improvement for core use cases

### After Phase 2 (Complex Filters)
- **29 filters** (+16 new filters, +123% increase from baseline)
- **Utilization:** 32% of available API capabilities
- **Estimated Implementation:** 5-7 days total (cumulative)
- **User Experience:** Comprehensive filtering for all major use cases

### After All Phases
- **60+ filters** (+47 new filters, +362% increase from baseline)
- **Utilization:** 70%+ of available API capabilities
- **Estimated Implementation:** 12-17 days total
- **User Experience:** Best-in-class D&D compendium filtering

---

## Next Steps

1. **Review this analysis** with stakeholder/user
2. **Prioritize phases** based on user needs and timeline
3. **Create implementation plan** for Phase 1 (quick wins)
4. **Design UI mockups** for new filter components
5. **Write tests first** (TDD mandate from CLAUDE.md)
6. **Implement Phase 1** with proper testing and commits
7. **Gather user feedback** before proceeding to Phase 2

---

## Appendix: API Filter Reference

### Common Filters (All Entities)
- `q` - Search query (Meilisearch)
- `page` - Pagination page number
- `per_page` - Results per page
- `sort_by` - Sort field
- `sort_direction` - Sort direction (asc/desc)
- `filter` - Advanced Meilisearch filter expression

### Spell-Specific Filters
- `level`, `school`, `concentration`, `ritual`, `damage_type`, `saving_throw`
- `requires_verbal`, `requires_somatic`, `requires_material`

### Item-Specific Filters
- `type`, `rarity`, `has_charges`, `has_prerequisites`, `spells`, `spells_operator`, `spell_level`, `min_strength`

### Monster-Specific Filters
- `challenge_rating`, `min_cr`, `max_cr`, `type`, `size`, `alignment`
- `spells`, `spells_operator`, `spell_level`, `spellcasting_ability`

### Race-Specific Filters
- `size`, `ability_bonus`, `min_speed`, `has_darkvision`
- `grants_proficiency`, `grants_skill`, `grants_proficiency_type`
- `speaks_language`, `language_choice_count`, `grants_languages`
- `has_innate_spells`, `spells`, `spells_operator`, `spell_level`

### Class-Specific Filters
- `base_only`, `is_spellcaster`, `hit_die`, `max_spell_level`
- `grants_proficiency`, `grants_skill`, `grants_saving_throw`
- `spells`, `spells_operator`, `spell_level`

### Background-Specific Filters
- `grants_proficiency`, `grants_skill`
- `speaks_language`, `language_choice_count`, `grants_languages`

### Feat-Specific Filters
- `has_prerequisites`, `prerequisite_ability`, `min_value`, `prerequisite_race`, `prerequisite_proficiency`
- `grants_proficiency`, `grants_skill`

---

**End of Analysis**
