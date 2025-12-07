# Frontend Refactoring Plan

**Created:** 2025-11-29
**Status:** Approved
**Estimated Effort:** ~13.5 hours total

## Overview

This plan identifies and prioritizes refactoring opportunities in the D&D 5e Compendium frontend. The codebase already follows strong patterns (filter factory, `useEntityList` composable), so these are **incremental improvements** rather than architectural overhauls.

## Current State Summary

| Layer | Count | Pattern |
|-------|-------|---------|
| Pages | 19 list, 7 detail | `useEntityList` + Pinia filter stores |
| Components | 135 Vue files | Entity-specific cards + reusable UI |
| Composables | 17 TypeScript files | Well-abstracted data fetching |
| Stores | 7 filter stores | Factory pattern (`createEntityFilterStore`) |

## Refactoring Opportunities

### 1. Reference Pages → UiListStates

**Priority:** 1 (Quick Win)
**Effort:** ~2 hours
**Impact:** ~300 lines saved

**Problem:** 9 reference pages have manual loading/error/empty state handling instead of using the existing `UiListStates` component.

**Pages to migrate:**
- `ability-scores/index.vue`
- `conditions/index.vue`
- `damage-types/index.vue`
- `item-types/index.vue`
- `languages/index.vue`
- `proficiency-types/index.vue`
- `sizes/index.vue`
- `skills/index.vue`
- `sources/index.vue`
- `spell-schools/index.vue`

**Before:**
```vue
<UiListSkeletonCards v-if="loading" />
<UiListErrorState v-else-if="error" ... />
<UiListEmptyState v-else-if="items.length === 0" ... />
<div v-else>
  <UiListResultsCount ... />
  <div class="grid ...">...</div>
</div>
```

**After:**
```vue
<UiListStates
  :loading="loading"
  :error="error"
  :empty="items.length === 0"
  :total="totalResults"
  entity-name="item"
  ...
>
  <template #grid>...</template>
</UiListStates>
```

---

### 2. Centralized Filter Options

**Priority:** 2
**Effort:** ~3 hours
**Impact:** Better organization, enables future i18n

**Problem:** 26+ hardcoded option arrays scattered across page components.

**Solution:** Create `app/config/filterOptions.ts` with exported constants:

```typescript
// Spell levels, CR options, range presets, movement types, etc.
export const SPELL_LEVEL_OPTIONS = [
  { label: 'Cantrip', value: '0' },
  // ...
] as const

export const CR_OPTIONS = [
  { label: 'CR 0', value: '0' },
  { label: 'CR 1/8', value: '0.125' },
  // ...
] as const

export const AC_RANGE_PRESETS = {
  'low': { label: 'Low (10-14)', min: 10, max: 14 },
  'medium': { label: 'Medium (15-17)', min: 15, max: 17 },
  'high': { label: 'High (18+)', min: 18, max: 25 },
} as const
```

---

### 3. Range Filter Helper

**Priority:** 3
**Effort:** ~1.5 hours
**Impact:** ~100 lines saved, cleaner page components

**Problem:** AC/HP/Cost range filters repeat manual `Record<string, string>` pattern.

**Solution:** Add `rangePreset` type to `useMeilisearchFilters`:

```typescript
// New filter configuration
{
  ref: selectedACRange,
  field: 'armor_class',
  type: 'rangePreset',
  presets: AC_RANGE_PRESETS
}

// Implementation in composable
case 'rangePreset':
  if (config.presets && value) {
    const preset = config.presets[value]
    if (preset) {
      filters.push(`${config.field} >= ${preset.min} AND ${config.field} <= ${preset.max}`)
    }
  }
  break
```

---

### 4. Base Entity Card Component

**Priority:** 4
**Effort:** ~7 hours
**Impact:** ~500 lines saved

**Problem:** 10+ entity cards repeat: `truncatedDescription`, `backgroundImage` pattern, card wrapper with hover effects, source footer.

**Solution:** Create `app/components/ui/card/UiEntityCard.vue`:

```vue
<UiEntityCard
  :to="`/spells/${spell.slug}`"
  entity-type="spells"
  :slug="spell.slug"
  color="spell"
  :description="spell.description"
  :sources="spell.sources"
>
  <template #badges>...</template>
  <template #title>...</template>
  <template #stats>...</template>
</UiEntityCard>
```

**Cards to migrate:**
- SpellCard, ItemCard, MonsterCard
- RaceCard, BackgroundCard, FeatCard, ClassCard
- LanguageCard, SizeCard
- Reference entity cards (AbilityScoreCard, ConditionCard, etc.)

---

## Implementation Order

| Phase | Task | Effort | GitHub Issue |
|-------|------|--------|--------------|
| 1 | Reference pages → UiListStates | 2h | [#32](https://github.com/dfox288/dnd-rulebook-project/issues/32) |
| 2 | Create `config/filterOptions.ts` | 1h | [#33](https://github.com/dfox288/dnd-rulebook-project/issues/33) |
| 3 | Add `rangePreset` to useMeilisearchFilters | 1.5h | [#34](https://github.com/dfox288/dnd-rulebook-project/issues/34) |
| 4 | Migrate pages to centralized options | 2h | (part of #33) |
| 5 | Create UiEntityCard base component | 2h | [#35](https://github.com/dfox288/dnd-rulebook-project/issues/35) |
| 6 | Migrate entity cards to base | 5h | (part of #35) |

---

## Success Criteria

- [ ] All reference pages use `UiListStates`
- [ ] Filter options centralized in `config/filterOptions.ts`
- [ ] `useMeilisearchFilters` supports `rangePreset` type
- [ ] `UiEntityCard` base component created
- [ ] At least 3 entity cards migrated to use base component
- [ ] All tests pass
- [ ] No visual regressions

---

## Notes

- Each phase is independently valuable and can be paused/resumed
- Follow TDD: write tests before implementation
- Commit after each logical unit of work
- The card refactoring (Phases 5-6) is optional if simpler wins satisfy goals
