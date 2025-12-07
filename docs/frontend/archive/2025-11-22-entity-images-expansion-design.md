# Entity Images Expansion - Design Document

**Date:** 2025-11-22
**Status:** 📋 Design Complete - Ready for Implementation
**Previous Feature:** Entity Images for Races (Complete)
**Goal:** Expand images to all remaining main entities (Spells, Items, Classes, Backgrounds, Feats)

---

## Overview

Expand the entity images feature from races to all 5 remaining main entity types, applying the same proven pattern: hero images (512px) on detail pages and subtle background images (256px) on list cards.

**Key Insight:** Infrastructure is already entity-agnostic. This is pure pattern replication, not new architecture.

---

## Design Principles

### 1. Reuse Existing Infrastructure
**No new components needed!**
- `useEntityImage` composable already supports all entity types
- `UiEntityHeaderWithImage` component is entity-agnostic
- Image path structure already includes all entities
- Docker volumes already mounted

### 2. Graceful Degradation
**Images optional, not required:**
- Pages work without images (text-only)
- No broken image placeholders
- `v-if` conditionals prevent rendering empty images
- Images appear automatically as they're generated

### 3. Sequential Implementation
**One entity at a time:**
- Lower risk (catch issues early)
- TDD per entity (RED-GREEN-REFACTOR)
- Incremental browser verification
- Clear git history (one commit per entity)

### 4. Strict TDD Mandate
**Tests before code, always:**
- Write 3 failing tests for card component
- Verify tests fail (RED)
- Implement minimal changes (GREEN)
- Run tests again, verify pass
- Commit immediately

---

## Architecture

### Existing Infrastructure (No Changes Needed)

**Composable:**
```typescript
useEntityImage(slug, entityType, size)
// entityType: 'spells' | 'items' | 'races' | 'classes' | 'backgrounds' | 'feats'
// size: '256' | '512' | 'original'
// Returns: string | null
```

**Component:**
```vue
<UiEntityHeaderWithImage
  :title="entity.name"
  :badges="badges"
  :image-path="imagePath"
  :image-alt="`${entity.name} portrait`"
/>
```

**Image Paths:**
```
/images/generated/conversions/256/{entity}/{provider}/{slug}.png  (cards)
/images/generated/conversions/512/{entity}/{provider}/{slug}.png  (detail)
```

### Per-Entity Changes Required

**1. Card Component (`{Entity}Card.vue`)**
- Add `backgroundImage` computed property
- Wrap template in `group` class
- Add background layer div with `v-if`
- Add z-index to content layer

**2. Detail Page (`{entity}/[slug].vue`)**
- Add `imagePath` computed property
- Replace `UiDetailPageHeader` with `UiEntityHeaderWithImage`
- Pass image path and alt text

**3. Card Tests (`{Entity}Card.test.ts`)**
- Add 3 tests: background exists, opacity classes, transitions

---

## Implementation Sequence

**Order (by complexity):**
1. **Spells** - Most similar to races (school/level badges)
2. **Items** - Complex card (rarity, magic, attunement)
3. **Classes** - Moderate (hit die, primary ability)
4. **Backgrounds** - Simpler structure
5. **Feats** - Simplest (prerequisites only)

**Rationale:** Start with most complex to validate pattern works for harder cases first.

---

## Per-Entity Workflow (6 Steps)

### Step 1: Read Existing Files
- Card component (`app/components/{entity}/{Entity}Card.vue`)
- Detail page (`app/pages/{entity}/[slug].vue`)
- Card tests (`tests/components/{entity}/{Entity}Card.test.ts`)

### Step 2: Add Failing Tests (RED Phase)
```typescript
// tests/components/{entity}/{Entity}Card.test.ts
it('renders background image when available', async () => {
  const wrapper = await mountSuspended({Entity}Card, { props: { {entity}: mock{Entity} } })
  const bgDiv = wrapper.find('[data-test="card-background"]')
  expect(bgDiv.exists()).toBe(true)
  expect(bgDiv.attributes('style')).toContain('background-image')
})

it('has correct opacity classes for background', async () => {
  const wrapper = await mountSuspended({Entity}Card, { props: { {entity}: mock{Entity} } })
  const bgDiv = wrapper.find('[data-test="card-background"]')
  expect(bgDiv.classes()).toContain('opacity-10')
  expect(bgDiv.classes()).toContain('group-hover:opacity-20')
})

it('applies transition to background opacity', async () => {
  const wrapper = await mountSuspended({Entity}Card, { props: { {entity}: mock{Entity} } })
  const bgDiv = wrapper.find('[data-test="card-background"]')
  expect(bgDiv.classes()).toContain('transition-opacity')
})
```

**Run tests, verify they FAIL:**
```bash
docker compose exec nuxt npm run test -- {Entity}Card
# Expected: 3 new tests fail
```

### Step 3: Update Card Component (GREEN Phase)

**Add computed property:**
```typescript
/**
 * Get background image path (256px variant)
 */
const backgroundImage = computed(() => {
  return useEntityImage(props.{entity}.slug, '{entity}s', '256')
})
```

**Update template:**
```vue
<template>
  <NuxtLink
    :to="`/{entity}s/${{entity}.slug}`"
    class="block h-full group"
  >
    <UCard class="relative overflow-hidden hover:shadow-lg transition-shadow h-full border border-gray-200 dark:border-gray-700">
      <!-- Background Image Layer -->
      <div
        v-if="backgroundImage"
        data-test="card-background"
        class="absolute inset-0 bg-cover bg-center opacity-10 transition-opacity duration-300 group-hover:opacity-20"
        :style="{ backgroundImage: `url(${backgroundImage})` }"
      />

      <!-- Content Layer -->
      <div class="relative z-10 flex flex-col h-full">
        <!-- existing content unchanged -->
      </div>
    </UCard>
  </NuxtLink>
</template>
```

**Run tests, verify they PASS:**
```bash
docker compose exec nuxt npm run test -- {Entity}Card
# Expected: All tests pass (including 3 new tests)
```

### Step 4: Update Detail Page

**Add computed property:**
```typescript
/**
 * Get entity image path (512px variant)
 */
const imagePath = computed(() => {
  if (!{entity}.value) return null
  return useEntityImage({entity}.value.slug, '{entity}s', '512')
})
```

**Replace header component:**
```vue
<!-- Replace UiDetailPageHeader with: -->
<UiEntityHeaderWithImage
  :title="{entity}.name"
  :badges="badges"
  :image-path="imagePath"
  :image-alt="`${{entity}.name} {entity-type} illustration`"
/>
```

### Step 5: Verify in Browser
```bash
# Restart dev server
docker compose restart nuxt && sleep 5

# Test URLs
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/{entity}s
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/{entity}s/{example-slug}

# Expected: Both return 200
```

### Step 6: Commit
```bash
git add app/components/{entity}/{Entity}Card.vue \
        app/pages/{entity}/[slug].vue \
        tests/components/{entity}/{Entity}Card.test.ts

git commit -m "feat: Add images to {entity} pages (TDD)

- Subtle background images on {entity} cards (256px, 10% opacity, 20% hover)
- Hero images on {entity} detail pages (512px, right-aligned)
- 3 new tests (all passing)
- Graceful degradation for missing images

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

**Repeat Steps 1-6 for next entity.**

---

## Testing Strategy

### Per-Entity Tests
**Card Component (3 new tests):**
1. Background image div exists with correct data-test attribute
2. Opacity classes applied (10% base, 20% hover)
3. Transition classes applied

**Detail Page (0 new tests):**
- No new tests needed (UiEntityHeaderWithImage already has 7 tests)
- Integration verified via browser testing

### Full Suite Verification
**After each entity:**
- Run full test suite: `docker compose exec nuxt npm run test`
- Verify no regressions
- Expected progression: 630 → 633 → 636 → 639 → 642 → 645 tests

### Browser Verification
**Per entity, test:**
- List page loads (HTTP 200)
- Detail page loads (HTTP 200)
- Images visible (if available)
- No broken image icons
- Hover effects work on cards
- Mobile responsive (image stacks on detail)

---

## Placeholder Handling

### Missing Images
**Graceful degradation built-in:**
- `useEntityImage` returns `null` for non-existent images
- Card: `v-if="backgroundImage"` prevents empty div
- Header: `v-if="imagePath"` prevents empty image element
- No console errors, no broken placeholders

### Partial Image Coverage
**Per-slug handling:**
- Entity A has image → Shows image
- Entity B no image → Shows text-only
- Both work correctly on same list page
- No layout shifts (absolute positioning)

### Image Loading States
**Browser-native handling:**
- Lazy loading via NuxtImg
- No custom loading spinners needed
- CSS background fails silently if 404
- Progressive enhancement approach

---

## Success Criteria

**Before marking complete:**
- [ ] All 5 entities implemented (spells, items, classes, backgrounds, feats)
- [ ] 15 new tests passing (3 per entity × 5)
- [ ] Total test suite: 645 tests passing
- [ ] ESLint clean (0 errors)
- [ ] TypeScript: No new errors
- [ ] Browser verified: 10+ pages (2 per entity × 5)
- [ ] 5 commits created (one per entity)
- [ ] CHANGELOG updated
- [ ] CURRENT_STATUS updated
- [ ] Handover document created

---

## File Manifest

### Files to Modify (15 total)

**Spells (3 files):**
- `app/components/spell/SpellCard.vue`
- `app/pages/spells/[slug].vue`
- `tests/components/spell/SpellCard.test.ts`

**Items (3 files):**
- `app/components/item/ItemCard.vue`
- `app/pages/items/[slug].vue`
- `tests/components/item/ItemCard.test.ts`

**Classes (3 files):**
- `app/components/class/ClassCard.vue`
- `app/pages/classes/[slug].vue`
- `tests/components/class/ClassCard.test.ts`

**Backgrounds (3 files):**
- `app/components/background/BackgroundCard.vue`
- `app/pages/backgrounds/[slug].vue`
- `tests/components/background/BackgroundCard.test.ts`

**Feats (3 files):**
- `app/components/feat/FeatCard.vue`
- `app/pages/feats/[slug].vue`
- `tests/components/feat/FeatCard.test.ts`

### Files to Create (1 file)
- `docs/HANDOVER-2025-11-22-ENTITY-IMAGES-EXPANSION.md`

### Files to Update (2 files)
- `CHANGELOG.md`
- `docs/CURRENT_STATUS.md`

---

## Risk Analysis

### Low Risk Items ✅
- **Proven pattern** - Already working for races
- **Reusable components** - No new code, just integration
- **Graceful degradation** - Missing images handled
- **Incremental commits** - Easy rollback per entity

### Medium Risk Items ⚠️
- **Entity-specific layouts** - Each card has unique structure (mitigated: pattern is flexible)
- **Test data variations** - Mock data differs per entity (mitigated: tests are simple)

### Mitigation Strategy
- Start with most complex entity (spells) to validate pattern
- Full test suite after each entity catches regressions early
- Browser verification confirms visual correctness
- Sequential approach allows stopping if issues arise

---

## Future Enhancements

**Easy to add later:**
1. **Monsters** - 7th entity type (same pattern)
2. **Image lightbox** - Click to view original size
3. **Fallback images** - Default placeholder per entity type
4. **WebP variants** - Add .webp alongside .png
5. **Different providers** - Switch via env variable

**Pattern is proven and extensible.**

---

## Estimated Timeline

**Per entity:** ~30-40 minutes
- Read files: 5 min
- Add tests: 5 min
- Update card: 10 min
- Update detail: 5 min
- Verify: 5 min
- Commit: 2 min

**Total for 5 entities:** ~2.5-3 hours

**Plus documentation:** ~30 min
**Grand total:** ~3-3.5 hours

---

**Status:** 📋 Design complete. Ready for implementation planning.
