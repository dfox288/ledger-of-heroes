# Handover Document - 2025-01-21 Session

## Session Summary

This session focused on enhancing the spell, item, and race pages with consistent design patterns, proper pagination, semantic colors, and complete data display.

## What Was Accomplished

### ✅ Spell Pages (Complete)
**List Page (`app/pages/spells/index.vue`):**
- Fixed pagination with NuxtUI v4 API (`v-model:page`, `items-per-page`)
- Added URL initialization from query params (page, search, filters)
- Implemented bidirectional URL sync for shareable filtered URLs
- Watch individual filter refs instead of computed queryParams

**Card Component (`app/components/spell/SpellCard.vue`):**
- Applied semantic NuxtUI v4 colors (primary, info, warning, error, success, neutral)
- Progressive level colors: Cantrip→1-3→4-6→7-9 (primary→info→warning→error)
- School color mapping by magic type
- Badge alignment: level left, school right (justify-between)
- Moved concentration/ritual badges into stats row (size sm)
- Added sources at bottom with full names and pages
- Used variant="subtle" for softer badge appearance

**Detail Page (`app/pages/spells/[slug].vue`):**
- Separate level and school badges with semantic colors
- Reduced header sizes: h1 (4xl), h2 (lg) for better hierarchy
- Replaced individual cards with UAccordion (type="multiple")
- Kept Description always visible
- Accordion sections: Higher Levels, Damage, Classes, Source
- Added all missing data fields (higher_levels, classes)

### ✅ Item Pages (Complete)
**List Page (`app/pages/items/index.vue`):**
- Fixed pagination (same as spells)
- URL initialization and bidirectional sync
- Individual ref watching

**Card Component (`app/components/item/ItemCard.vue`):**
- Semantic rarity colors: common→uncommon→rare→very rare→legendary→artifact
  (neutral→success→info→primary→warning→error)
- Semantic type colors by category:
  - Weapons: error (red)
  - Armor: info (blue)
  - Tools: warning (amber)
  - Potions: success (green)
  - Wondrous Items: primary (teal)
- Type left, rarity right alignment
- Magic/attunement badges in stats row (size sm)
- Cost display with gold/copper formatting
- Sources at bottom
- Rarity text capitalization ("Very Rare" not "very rare")

**Detail Page (`app/pages/items/[slug].vue`):**
- Same semantic colors for type and rarity
- Smaller headers (h1: 4xl, h2: lg)
- Accordion: Properties, Modifiers, Abilities, Source
- Description always visible

### ✅ Race Pages (Complete)
**List Page (`app/pages/races/index.vue`):**
- Fixed pagination
- URL initialization and sync
- Header format matching spells/items: "Races (X total)"

**Card Component (`app/components/race/RaceCard.vue`):**
- Size-based semantic colors:
  - Small: success (green)
  - Medium: info (blue)
  - Large: warning (amber)
  - Huge/Gargantuan: error (red)
- Size left, race/subrace right alignment
- Fixed subrace badge logic (checks subraces array presence)
- Subrace count badge ("🌟 4 Subraces")
- Traits badge in stats row
- Sources at bottom

**Detail Page (`app/pages/races/[slug].vue`):**
- Size badge with semantic color
- Race/Subrace badge (info/primary)
- Smaller headers
- Accordion: Parent Race, Subraces, Traits, Modifiers, Languages, Proficiencies, Spells, Conditions, Source
- Description and Ability Score Increases always visible
- All API fields now displayed (parent_race, subraces, modifiers, spells, conditions)

## Key Design Patterns Established

### Pagination Pattern (NuxtUI v4)
```vue
<UPagination
  v-model:page="currentPage"
  :total="totalResults"
  :items-per-page="perPage"
  show-edges
/>
```
- Conditional: `v-if="totalResults > perPage"`
- Watch individual refs: `[currentPage, searchQuery, ...]`
- Initialize from URL: `ref(route.query.page ? Number(route.query.page) : 1)`

### Semantic Color Pattern
**NuxtUI v4 colors:** `primary`, `info`, `warning`, `error`, `success`, `neutral`
- Use for progressive scales (levels, rarity)
- Use for category coding (item types, sizes)
- Apply with `variant="subtle"` for badges

### Card Layout Pattern
```vue
<div class="flex flex-col h-full">
  <div class="space-y-3 flex-1">
    <!-- Top badges (left/right alignment) -->
    <!-- Name -->
    <!-- Stats row with badges -->
    <!-- Description -->
  </div>
  <!-- Sources (bottom separator) -->
</div>
```

### Detail Page Pattern
- Keep most important data always visible (Description, primary stats)
- Use UAccordion with `type="multiple"` for supplementary info
- Smaller headers: h1 (text-4xl), h2 (text-lg)
- Reduce text: description (text-base, leading-relaxed)

## Remaining Work

### 🔜 Classes Pages (Not Started)
Apply same pattern:
- Fix pagination with URL support
- Create/enhance ClassCard component with semantic colors
- Add accordion to detail page
- Ensure all API fields displayed
- **Color strategy:** TBD - options:
  - Class role (martial=error, caster=primary, hybrid=warning)?
  - Hit die size?
  - User should decide

### 🔜 Backgrounds Pages (Not Started)
Apply same pattern:
- Fix pagination
- Enhance BackgroundCard
- Add accordion
- Display all fields
- **Color strategy:** TBD - backgrounds are simpler, may use neutral/info

### 🔜 Feats Pages (Not Started)
Apply same pattern:
- Fix pagination
- Enhance FeatCard
- Add accordion
- Display all fields
- **Color strategy:** TBD - perhaps by feat type/category if available

## Technical Notes

### NuxtUI v4 Breaking Changes
- Badge colors: Use semantic names (primary, info, etc.) NOT Tailwind names (blue, red)
- UPagination: `v-model:page` (not `v-model`), `items-per-page` (not `page-count`)
- USelectMenu: `:items` (not `:options`), `value-key` prop required

### Common Issues Fixed
1. **Pagination not working:** Wrong v-model syntax or watching computed instead of refs
2. **Badges no color:** Using Tailwind color names instead of semantic
3. **Accordion not showing:** Wrong conditional (`lastPage > 1` instead of `totalResults > perPage`)

## File Locations

### Components
- `app/components/spell/SpellCard.vue`
- `app/components/item/ItemCard.vue`
- `app/components/race/RaceCard.vue`
- `app/components/class/ClassCard.vue` (needs work)
- `app/components/background/BackgroundCard.vue` (needs work)
- `app/components/feat/FeatCard.vue` (needs work)

### Pages
- `app/pages/{entity}/index.vue` - List pages
- `app/pages/{entity}/[slug].vue` - Detail pages

### Server API Proxies
- `server/api/{entity}/index.get.ts` - List proxy
- `server/api/{entity}/[slug].get.ts` - Detail proxy
- `server/api/{entity}-{lookup}.get.ts` - Lookup data (types, schools, etc.)

## Git Commits This Session

```bash
git log --oneline -10
```

Recent commits:
- `b999bb5` fix: Fix race page issues and add all available data fields
- `89135d4` feat: Apply spell/item enhancements to race pages
- `df8eb90` fix: Capitalize rarity text on item detail page
- `ad33bd3` feat: Add semantic color coding for item types and rarity
- `6777b41` feat: Enhance ItemCard with spell card design pattern
- `85d688e` feat: Apply spell page enhancements to item list and detail pages
- `f75a7bf` feat: Enhance spell detail page with badges, accordion, and all data fields
- `cb56cb7` fix: Fix spell pagination and improve badge styling

## Questions for User (If Continuing)

Before implementing Classes, Backgrounds, and Feats:

1. **Class Color Strategy:** What semantic colors for classes?
   - By role (martial/caster/hybrid)?
   - By hit die?
   - By primary stat?

2. **Background Color Strategy:** Backgrounds are simpler - single color or variation?

3. **Feat Color Strategy:** Color by feat type/category, or single color?

4. **Priority Order:** Which entity type to complete next (Classes, Backgrounds, or Feats)?

## Handover Checklist

- ✅ CLAUDE.md updated with latest status
- ✅ Handover document created
- ✅ All changes committed to git
- ✅ Patterns documented
- ✅ Remaining work identified
- ✅ Questions for next session noted

## For Next Agent

Start by:
1. Reading this handover document
2. Asking user which entity type to work on next
3. Asking user about color strategy for that entity type
4. Following the established patterns from spells/items/races
5. Ensuring all API fields are displayed
6. Testing pagination works with URL support
