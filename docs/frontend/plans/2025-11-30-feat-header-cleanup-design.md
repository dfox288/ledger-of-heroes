# Feat Detail Page Header Cleanup Design

**Date:** 2025-11-30
**Issue:** #68
**Status:** Approved

## Problem

The feat detail page has poor information density for feats with minimal data (like Fey Touched):
- Empty prerequisites column wastes 2/3 width when no prerequisites exist
- Image isolated in narrow 1/3 column
- "What You Get" section: often just 1 card in a 3-column grid
- "Granted Spells" section: often just 1-2 cards in a 3-column grid
- Result: excessive vertical scrolling, sparse-looking page

## Solution

Create a unified "hero section" that combines benefits, granted spells, and image in a side-by-side layout.

## New Page Structure

```
1. Breadcrumb
2. Header (title + badges including +N ABILITY)
3. Prerequisites (full width, only if present)
4. Hero Section (2-column grid):
   ├── Left (2/3): Benefits + Granted Spells (stacked vertically)
   └── Right (1/3): Image
5. Description (full width)
6. Related Variants
7. Accordions (Source, Tags)
8. Bottom Nav
```

## Hero Section Layout

```vue
<div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
  <!-- Left: Benefits + Spells -->
  <div class="lg:col-span-2 space-y-6">
    <FeatBenefitsGrid v-if="hasBenefits" ... />
    <FeatGrantedSpells v-if="hasSpells" ... />
  </div>

  <!-- Right: Image -->
  <div class="lg:col-span-1">
    <UiDetailEntityImage ... />
  </div>
</div>
```

**Responsive:**
- Desktop (lg+): 2/3 benefits | 1/3 image
- Mobile/Tablet: Stacked (benefits → spells → image)

## Component Changes

### FeatBenefitsGrid
- Change grid from `lg:grid-cols-3` to `md:grid-cols-2 lg:grid-cols-2`
- Max 2 columns when inside constrained hero width

### FeatGrantedSpells
- Change grid from `lg:grid-cols-3` to `md:grid-cols-2 lg:grid-cols-2`
- Max 2 columns when inside constrained hero width

### No Changes Needed
- `UiDetailEntityImage`
- `FeatVariantsSection`
- `UiDetailDescriptionCard`
- Accordions

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| No benefits, no spells | Image in 1/3 column, left side empty (consistent with other pages) |
| Benefits only | Benefits + Image side-by-side |
| Spells only | Spells + Image side-by-side |
| Many benefits (3+) | 2-column grid wraps, image fixed |
| Prerequisites present | Full width above hero section |

## Design Decisions

1. **Keep ability modifier in both places** - Header badge for quick reference, benefits card for detail
2. **Image stays in 1/3 column even when no benefits** - Consistency with other detail pages
3. **Max 2 columns for benefit/spell cards** - Prevents cramped cards in constrained width

## Files to Modify

1. `app/pages/feats/[slug].vue` - Restructure layout
2. `app/components/feat/BenefitsGrid.vue` - Adjust grid columns
3. `app/components/feat/GrantedSpells.vue` - Adjust grid columns

## Test Cases

- [ ] Fey Touched (Intelligence): Benefits + Spells + Image side-by-side
- [ ] Actor: Multiple benefits cards wrap in 2-column grid
- [ ] Lucky: No prerequisites, no benefits - image in 1/3 column
- [ ] War Caster: Prerequisites show above hero section
- [ ] Mobile: All sections stack vertically
