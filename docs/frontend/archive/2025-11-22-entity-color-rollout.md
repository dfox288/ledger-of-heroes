# Entity Color System Rollout Plan

**Date:** 2025-11-22
**Status:** Approved - Ready for Implementation
**Approach:** Incremental Rollout by Entity Type

---

## Overview

Roll out the newly implemented entity color system across all components and pages in the D&D compendium. Replace hardcoded generic colors (`primary`, `info`, `warning`, etc.) with semantic entity-specific colors (`spell`, `item`, `race`, etc.) to create instant visual recognition of entity types.

## Goals

1. **Visual Recognition** - Users can identify entity types by color instantly
2. **Consistent Branding** - All spell-related UI uses arcane purple, all items use treasure gold, etc.
3. **Simplified Code** - Remove complex color mapping functions in favor of direct semantic color usage
4. **Maintainability** - Centralize color definitions in config files, not scattered across components

## Current State

### What Exists
- ✅ Entity color system fully implemented (3-layer config in nuxt.config.ts → app.config.ts → main.css)
- ✅ 17 semantic entity colors registered and working
- ✅ Color test page validates all colors render correctly

### What Needs Work
- ❌ **18 entity card components** still use old generic colors
- ❌ **23 page files** use hardcoded color values
- ❌ **SearchResultCard** uses color mapping instead of semantic colors
- ❌ **Search filter buttons** use generic colors
- ❌ **12 test files** have color assertions expecting old colors

**Total impact: ~53 files**

## Design Principle: Zero Abstraction

**Key insight:** The three-layer color system already handles all mapping. No composables, no utility functions, no abstraction needed.

**How it works:**
1. `nuxt.config.ts` registers `'spell'` as a valid NuxtUI color
2. `app.config.ts` maps `spell: 'arcane'`
3. NuxtUI resolves `color="spell"` → `arcane` palette automatically
4. **We just use entity names directly in templates!**

### Before & After Examples

**SpellCard.vue - BEFORE:**
```vue
<script setup lang="ts">
const getLevelColor = (level: number) => {
  if (level === 0) return 'primary'
  if (level <= 3) return 'info'
  if (level <= 6) return 'warning'
  return 'error'
}
</script>

<template>
  <UBadge :color="getLevelColor(spell.level)">
    {{ levelText }}
  </UBadge>
</template>
```

**SpellCard.vue - AFTER:**
```vue
<template>
  <!-- Just use the literal string 'spell' -->
  <UBadge color="spell">
    {{ levelText }}
  </UBadge>
</template>
```

**SearchResultCard.vue - BEFORE:**
```vue
<script setup lang="ts">
const getBadgeColor = computed(() => {
  const colors: Record<string, ColorName> = {
    spell: 'primary',
    item: 'warning',
    race: 'info',
    class: 'error',
    background: 'success',
    feat: 'warning'
  }
  return colors[props.type] || 'neutral'
})
</script>

<template>
  <UBadge :color="getBadgeColor">{{ type }}</UBadge>
</template>
```

**SearchResultCard.vue - AFTER:**
```vue
<template>
  <!-- props.type is already 'spell', 'item', etc. - just use it! -->
  <UBadge :color="type">{{ type }}</UBadge>
</template>
```

**No mapping function needed at all!**

## Implementation Strategy: Incremental by Entity

### Approach: Incremental Rollout (Recommended)

**Why incremental:**
- ✅ Lower risk - test each entity independently
- ✅ Easy to spot issues - only one entity type changes at a time
- ✅ Can ship incrementally - merge after each entity
- ✅ Rollback is surgical - just one entity, not everything
- ✅ User sees gradual improvement

**Why NOT big-bang:**
- ❌ High risk - one mistake breaks everything
- ❌ Harder to review all changes at once
- ❌ Can't test incrementally
- ❌ Rollback means losing all work

### Rollout Order

**Phase A: Main Entity Cards (7 entities)**
Update in this order (most complex → simplest):
1. SpellCard.vue
2. ItemCard.vue
3. RaceCard.vue
4. ClassCard.vue
5. BackgroundCard.vue
6. FeatCard.vue
7. MonsterCard.vue

**Phase B: Reference Entity Cards (10 entities)**
Update alphabetically:
1. AbilityScoreCard.vue
2. ConditionCard.vue
3. DamageTypeCard.vue
4. ItemTypeCard.vue
5. LanguageCard.vue
6. ProficiencyTypeCard.vue
7. SizeCard.vue
8. SkillCard.vue
9. SpellSchoolCard.vue
10. SourceCard.vue

**Phase C: Search Components**
1. SearchResultCard.vue
2. app/pages/search.vue (filter buttons)

**Phase D: Shared UI Components (as needed)**
Only update if they directly display entity types:
- UiTagsDisplay.vue
- UiAccordion* components
- Error/Empty states (probably keep generic)

## Refactoring Checklist (Per Entity)

For each entity card, follow this TDD process:

### 1. Write Test First (RED)
```typescript
// tests/components/spell/SpellCard.test.ts
it('uses spell entity color for badge', async () => {
  const wrapper = await mountSuspended(SpellCard, {
    props: { spell: mockSpell }
  })

  const badge = wrapper.find('[data-test="entity-badge"]')
  expect(badge.attributes('color')).toBe('spell')
})
```

### 2. Update Component (GREEN)
```vue
<!-- Remove color mapping functions from <script> -->
<!-- Replace in <template>: -->
- <UBadge color="primary">        ❌ OLD
+ <UBadge color="spell">          ✅ NEW

- :color="getLevelColor(level)"  ❌ OLD
+ color="spell"                   ✅ NEW
```

### 3. Update Additional Tests
Search for color assertions in the test file:
```typescript
- expect(color).toBe('primary')   ❌ OLD
+ expect(color).toBe('spell')     ✅ NEW
```

### 4. Manual Verification
- Visit entity list page (e.g., `/spells`)
- Verify entity color appears correctly
- Toggle dark mode - verify colors work
- Hover over cards - verify borders/hover states

### 5. Commit
```bash
git add app/components/spell/ tests/components/spell/
git commit -m "refactor: Apply spell entity colors to SpellCard

- Replace generic color mapping with semantic 'spell' color
- Update tests to expect 'spell' color
- Simplify component by removing getLevelColor() function
- Verified in browser: purple theme applied correctly

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

### 6. Repeat for Next Entity

## Specific Component Patterns

### Pattern 1: Simple Badge Replacement
**Before:**
```vue
<UBadge color="primary">{{ name }}</UBadge>
```

**After:**
```vue
<UBadge color="spell">{{ name }}</UBadge>
```

### Pattern 2: Remove Computed Color Functions
**Before:**
```vue
<script setup lang="ts">
const badgeColor = computed(() => {
  return level > 5 ? 'error' : 'info'
})
</script>
<template>
  <UBadge :color="badgeColor">Level {{ level }}</UBadge>
</template>
```

**After:**
```vue
<template>
  <!-- Use entity color consistently, don't vary by level -->
  <UBadge color="spell">Level {{ level }}</UBadge>
</template>
```

### Pattern 3: Borders & Tailwind Classes
**Before:**
```vue
<UCard class="border border-gray-200 dark:border-gray-700">
```

**After:**
```vue
<UCard class="border-2 border-spell-300 dark:border-spell-700 hover:border-spell-500">
```

**Intensity guidelines:**
- Light mode border: `300` (subtle)
- Dark mode border: `700` (visible)
- Hover state: `500` (base color)

### Pattern 4: Dynamic Entity Types (SearchResultCard)
**Before:**
```vue
<script setup lang="ts">
interface Props {
  type: 'spell' | 'item' | 'race' | ...
}

const getBadgeColor = computed(() => {
  const colors = { spell: 'primary', item: 'warning', ... }
  return colors[props.type]
})
</script>
<template>
  <UBadge :color="getBadgeColor">{{ type }}</UBadge>
</template>
```

**After:**
```vue
<script setup lang="ts">
interface Props {
  type: 'spell' | 'item' | 'race' | ...
}
// No computed function needed!
</script>
<template>
  <UBadge :color="type">{{ type }}</UBadge>
</template>
```

### Pattern 5: Sub-Categorization Colors (Keep Dynamic)
Some badges should remain dynamic for sub-categorization:

```vue
<!-- KEEP dynamic for spell schools, item rarity, etc. -->
<UBadge :color="getSchoolColor(school.code)" variant="soft">
  {{ school.name }}
</UBadge>

<!-- But main entity badge uses entity color -->
<UBadge color="spell" variant="subtle">
  {{ levelText }}
</UBadge>
```

**Guideline:**
- **Entity type badge** → Static entity color (`color="spell"`)
- **Sub-category badge** → Dynamic mapping (`getSchoolColor()`, `getRarityColor()`)

## Testing Strategy

### Test File Updates (12 files)

Most test updates are simple find-replace:

```typescript
// BEFORE
expect(badge.attributes('color')).toBe('primary')

// AFTER
expect(badge.attributes('color')).toBe('spell')
```

### Running Tests

After each entity update:
```bash
# Run tests for that entity
npm run test -- SpellCard.test.ts

# Run full suite before committing
npm run test
```

### Test Coverage Goals
- ✅ All color assertions updated to expect entity colors
- ✅ All tests pass
- ✅ No new test failures introduced
- ✅ Coverage percentage maintained or improved

## Visual Verification Checklist

For each entity, manually verify:

- [ ] List page shows entity color (e.g., `/spells` shows purple)
- [ ] Card borders use entity color
- [ ] Hover states enhance entity color
- [ ] Detail page badges use entity color
- [ ] Light mode looks good (colors vibrant enough)
- [ ] Dark mode looks good (colors not overwhelming)
- [ ] Search results show correct entity color
- [ ] Search filters show correct entity color when active

## Rollback Plan

If an entity's colors look wrong:

1. **Identify issue** - Which entity? What's wrong?
2. **Quick fix** - If it's a simple fix (wrong intensity), fix it
3. **Revert commit** - If it's a design issue, revert that entity's commit
4. **Adjust design** - Update color palette in main.css
5. **Re-apply** - Retry the entity after fixing color definition

## Success Criteria

### Phase A Complete (Main Entities)
- [x] All 7 main entity cards use entity-specific colors
- [x] All entity tests pass
- [x] All entity list pages show correct colors
- [x] Light and dark modes verified for all entities

### Phase B Complete (Reference Entities)
- [x] All 10 reference entity cards use entity-specific colors
- [x] All reference entity tests pass
- [x] All reference pages show correct colors

### Phase C Complete (Search)
- [x] SearchResultCard uses dynamic entity colors
- [x] Search filter buttons show entity colors when active
- [x] Search page verified in browser

### Project Complete
- [x] All 53 files updated
- [x] All tests pass
- [x] No visual regressions
- [x] User can identify entity types by color alone
- [x] Code is simpler (fewer color mapping functions)
- [x] Design documentation updated
- [x] CHANGELOG.md updated

## Estimated Timeline

**Per entity card:** 15-30 minutes (following TDD)
- Write test: 5 min
- Update component: 10 min
- Verify in browser: 5 min
- Commit: 2 min

**Total estimates:**
- Phase A (7 entities): 2-3.5 hours
- Phase B (10 entities): 2.5-5 hours
- Phase C (search): 30-60 minutes
- Phase D (shared components): 1-2 hours

**Grand total: 6-11 hours** (can be split across multiple sessions)

## Notes

- Entity colors are **thematic identifiers**, not information hierarchy
- Don't vary entity color by severity/importance - keep it consistent
- Sub-categories (spell schools, item rarity) can still use dynamic colors
- Always test in both light and dark modes
- Commit frequently - one entity at a time

---

**Next Steps:**
1. Set up git worktree for isolated work
2. Create detailed implementation plan with bite-sized tasks
3. Begin Phase A: Main entity cards
