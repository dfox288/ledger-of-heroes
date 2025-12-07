# Reference Entity Images - Design Document

**Date:** 2025-11-22
**Status:** ✅ Approved
**Feature:** Extend entity images to all reference entities
**Type:** Enhancement

---

## 📋 Overview

Extend the entity images feature to support all 16 entity types (6 main + 10 reference) by refactoring the `useEntityImage` composable and adding background images to reference entity list cards.

### Goals

1. **Universal Image Support:** Enable all entity types to use AI-generated images
2. **Type Safety:** Maintain compile-time validation for entity types
3. **Naming Flexibility:** Handle kebab-case routes → snake_case image folders transparently
4. **Visual Consistency:** Apply established background image pattern to reference cards
5. **Backward Compatibility:** Existing main entity implementations continue working

### Non-Goals

- ❌ Detail pages for reference entities (future work)
- ❌ Custom image variants beyond 256/512/original
- ❌ Dynamic image provider switching per entity
- ❌ Image preloading or optimization (already handled by NuxtImg)

---

## 🎯 Problem Statement

### Current Limitations

**1. Composable Scope Limited:**
```typescript
// Current: Only 6 main entities supported
type Entity = 'races' | 'classes' | 'backgrounds' | 'feats' | 'spells' | 'items'
```

**2. Naming Mismatch:**
- **Frontend Routes:** `spell-schools`, `item-types`, `ability-scores` (kebab-case)
- **Image Folders:** `spell_schools`, `item_types`, `ability_scores` (snake_case)

**3. Reference Cards Missing Images:**
10 reference entity cards have no background images despite images being available in `../image-generator/output/`

### Available Resources

**Generated Images Exist For:**
- ✅ ability_scores (6 images)
- ✅ conditions (15 images)
- ✅ damage_types (13 images)
- ✅ item_types (20 images)
- ✅ languages (30 images)
- ✅ proficiency_types (40 images)
- ✅ sizes (6 images)
- ✅ skills (18 images)
- ✅ spell_schools (8 images)
- ✅ sources (8 images)

**Image Structure:**
```
../image-generator/output/
├── {entity}/stability-ai/{slug}.png           (original)
└── conversions/
    ├── 256/{entity}/stability-ai/{slug}.png   (thumbnail)
    └── 512/{entity}/stability-ai/{slug}.png   (hero)
```

---

## 🏗️ Architecture

### Component 1: Enhanced `useEntityImage` Composable

**New Signature:**
```typescript
export function useEntityImage() {
  return {
    getImagePath(
      entityType: EntityType,
      slug: string,
      size: ImageSize = '256'
    ): string | null
  }
}
```

**Type Definitions:**
```typescript
type EntityType =
  // Main entities (existing, 6 total)
  | 'races' | 'classes' | 'backgrounds' | 'feats' | 'spells' | 'items'
  // Reference entities (new, 10 total)
  | 'ability-scores' | 'conditions' | 'damage-types' | 'item-types'
  | 'languages' | 'proficiency-types' | 'sizes' | 'skills'
  | 'spell-schools' | 'sources'

type ImageSize = '256' | '512' | 'original'
```

**Entity-to-Folder Mapping:**
```typescript
const ENTITY_FOLDER_MAP: Record<EntityType, string> = {
  // Main entities (direct match)
  'races': 'races',
  'classes': 'classes',
  'backgrounds': 'backgrounds',
  'feats': 'feats',
  'spells': 'spells',
  'items': 'items',

  // Reference entities (conversion needed)
  'ability-scores': 'ability_scores',
  'spell-schools': 'spell_schools',
  'item-types': 'item_types',
  'proficiency-types': 'proficiency_types',
  'damage-types': 'damage_types',

  // Reference entities (direct match)
  'conditions': 'conditions',
  'languages': 'languages',
  'sizes': 'sizes',
  'skills': 'skills',
  'sources': 'sources',
}
```

**Implementation Logic:**
```typescript
export function useEntityImage() {
  const config = useRuntimeConfig()
  const provider = config.public.imageProvider

  return {
    getImagePath(
      entityType: EntityType,
      slug: string,
      size: ImageSize = '256'
    ): string | null {
      // Validation
      if (!slug || slug.trim() === '') return null
      if (!ENTITY_FOLDER_MAP[entityType]) return null

      // Map entity type to folder name
      const folderName = ENTITY_FOLDER_MAP[entityType]

      // Build path based on size
      if (size === 'original') {
        return `/images/generated/${folderName}/${provider}/${slug}.png`
      }

      return `/images/generated/conversions/${size}/${folderName}/${provider}/${slug}.png`
    }
  }
}
```

**Key Features:**
1. **Type-Safe:** Union type prevents invalid entity types at compile time
2. **Transparent Mapping:** Developers use route names, composable handles folder conversion
3. **Validated:** Empty slugs return null (graceful degradation)
4. **Provider-Agnostic:** Uses runtime config for provider (currently `stability-ai`)

---

### Component 2: Reference Entity Card Pattern

**Before (No Images):**
```vue
<template>
  <UCard class="hover:shadow-lg transition-shadow h-full">
    <div class="space-y-3">
      <!-- content -->
    </div>
  </UCard>
</template>
```

**After (With Background Images):**
```vue
<script setup lang="ts">
interface Props {
  skill: Skill
}

const props = defineProps<Props>()
const { getImagePath } = useEntityImage()
const backgroundImageUrl = computed(() =>
  getImagePath('skills', props.skill.slug, 256)
)
</script>

<template>
  <div
    :style="backgroundImageUrl ? {
      backgroundImage: `url(${backgroundImageUrl})`,
      backgroundSize: 'cover',
      backgroundPosition: 'center'
    } : {}"
    class="group relative overflow-hidden rounded-lg border border-default
           transition-all duration-200 hover:border-primary hover:scale-[1.02]
           hover:shadow-lg dark:hover:shadow-primary/20
           after:absolute after:inset-0 after:bg-background/90 hover:after:bg-background/80
           after:transition-colors after:duration-200"
  >
    <div class="relative z-10 p-4 space-y-3">
      <!-- existing card content -->
    </div>
  </div>
</template>
```

**Pattern Details:**
- **Background Image:** 256px variant, 10% opacity (90% overlay)
- **Hover Effect:** 20% opacity (80% overlay) for engagement
- **Positioning:** CSS background (cover, center)
- **Layering:** `after` pseudo-element for overlay, content at `z-10`
- **No Links:** Reference cards are display-only (no detail pages)

---

### Component 3: Migration Strategy

**Phase 1: Composable Refactor (TDD)**
1. Write tests for new signature (all 16 entity types)
2. Implement entity mapping logic
3. Verify backward compatibility
4. Update existing tests

**Phase 2: Update Main Entities (6 cards + 6 detail pages)**
Update existing usage to new signature:
```typescript
// OLD
const url = useEntityImage(props.race.slug, 'races', 256)

// NEW
const { getImagePath } = useEntityImage()
const url = computed(() => getImagePath('races', props.race.slug, 256))
```

**Phase 3: Update Reference Entities (10 cards)**
Add background images following established pattern.

**Phase 4: Testing & Verification**
- Full test suite: 645 + ~35 new = 680 tests
- Browser verification: All list pages HTTP 200
- Visual verification: Background images visible

---

## 📊 Data Flow

```
User visits /skills
    ↓
Skills list page loads
    ↓
SkillCard component renders (18 cards)
    ↓
Each card calls: getImagePath('skills', slug, 256)
    ↓
Composable maps: 'skills' → 'skills' (no conversion)
    ↓
Returns: /images/generated/conversions/256/skills/stability-ai/acrobatics.png
    ↓
Card applies as CSS background-image
    ↓
Missing images handled gracefully (no background shown)
```

**Docker Volume Mount:**
```yaml
# docker-compose.yml
volumes:
  - ../image-generator/output:/app/public/images/generated:ro
```

**Image Serving:**
- Development: Docker volume mount (read-only)
- Production: Static files served by Nginx

---

## 🧪 Testing Strategy

### Composable Tests

**File:** `tests/composables/useEntityImage.test.ts`

**Test Coverage:**
```typescript
describe('useEntityImage', () => {
  describe('getImagePath', () => {
    // Entity type validation (16 tests)
    it('returns path for each main entity type', () => { ... })
    it('returns path for each reference entity type', () => { ... })

    // Naming conversion (5 tests)
    it('converts ability-scores to ability_scores', () => { ... })
    it('converts spell-schools to spell_schools', () => { ... })
    it('converts item-types to item_types', () => { ... })
    it('converts proficiency-types to proficiency_types', () => { ... })
    it('converts damage-types to damage_types', () => { ... })

    // Direct matches (11 tests)
    it('uses direct match for skills', () => { ... })
    it('uses direct match for conditions', () => { ... })
    // ... etc

    // Size variants (3 tests)
    it('returns 256px conversion path', () => { ... })
    it('returns 512px conversion path', () => { ... })
    it('returns original path', () => { ... })

    // Validation (4 tests)
    it('returns null for empty slug', () => { ... })
    it('returns null for whitespace-only slug', () => { ... })
    it('returns null for invalid entity type', () => { ... })
    it('returns null for invalid size', () => { ... })
  })
})
```

**Total Composable Tests:** ~35 tests

### Card Component Tests

**Pattern for Each Reference Card:**
```typescript
describe('SkillCard', () => {
  // Existing tests ...

  // NEW: Image tests
  it('computes background image URL correctly', async () => {
    const wrapper = await mountSuspended(SkillCard, {
      props: { skill: { id: 1, slug: 'acrobatics', name: 'Acrobatics' } }
    })

    const url = wrapper.vm.backgroundImageUrl
    expect(url).toBe('/images/generated/conversions/256/skills/stability-ai/acrobatics.png')
  })

  it('applies background image styles when URL exists', async () => {
    const wrapper = await mountSuspended(SkillCard, {
      props: { skill: { id: 1, slug: 'acrobatics', name: 'Acrobatics' } }
    })

    const card = wrapper.find('.group')
    const style = card.attributes('style')
    expect(style).toContain('background-image')
    expect(style).toContain('acrobatics.png')
  })
})
```

**Cards to Test (10 files):**
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

**Total Card Tests:** 20 tests (2 per card)

**Grand Total:** ~55 new tests

---

## ⚠️ Error Handling

### Graceful Degradation

**Invalid Inputs:**
- Empty slug → returns `null` → no background image
- Invalid entity type → TypeScript compile error (prevented)
- Invalid size → returns `null` → no background image

**Missing Files:**
- Image file doesn't exist → browser handles gracefully → no background shown
- No visual error, card still renders perfectly

**Provider Issues:**
- Provider not configured → returns `null` → no background
- Provider folder missing → browser handles → no background

### No Breaking Changes

- Composable refactor maintains return type (`string | null`)
- All existing functionality preserved
- New features are additive only
- Missing images don't break UI

---

## 📁 File Changes

### Files to Modify (24 total)

**1. Composable (1 file)**
- `app/composables/useEntityImage.ts` - Refactor to new signature

**2. Composable Tests (1 file)**
- `tests/composables/useEntityImage.test.ts` - Add ~35 tests

**3. Main Entity Cards (6 files)**
- `app/components/race/RaceCard.vue`
- `app/components/class/ClassCard.vue`
- `app/components/background/BackgroundCard.vue`
- `app/components/feat/FeatCard.vue`
- `app/components/spell/SpellCard.vue`
- `app/components/item/ItemCard.vue`

**4. Main Entity Detail Pages (6 files)**
- `app/pages/races/[slug].vue`
- `app/pages/classes/[slug].vue`
- `app/pages/backgrounds/[slug].vue`
- `app/pages/feats/[slug].vue`
- `app/pages/spells/[slug].vue`
- `app/pages/items/[slug].vue`

**5. Reference Entity Cards (10 files)**
- `app/components/ability-score/AbilityScoreCard.vue`
- `app/components/condition/ConditionCard.vue`
- `app/components/damage-type/DamageTypeCard.vue`
- `app/components/item-type/ItemTypeCard.vue`
- `app/components/language/LanguageCard.vue`
- `app/components/proficiency-type/ProficiencyTypeCard.vue`
- `app/components/size/SizeCard.vue`
- `app/components/skill/SkillCard.vue`
- `app/components/spell-school/SpellSchoolCard.vue`
- `app/components/source/SourceCard.vue`

### Files to Create (10 test files)

**Reference Card Tests:**
- `tests/components/ability-score/AbilityScoreCard.test.ts`
- `tests/components/condition/ConditionCard.test.ts`
- `tests/components/damage-type/DamageTypeCard.test.ts`
- `tests/components/item-type/ItemTypeCard.test.ts`
- `tests/components/language/LanguageCard.test.ts`
- `tests/components/proficiency-type/ProficiencyTypeCard.test.ts`
- `tests/components/size/SizeCard.test.ts`
- `tests/components/skill/SkillCard.test.ts`
- `tests/components/spell-school/SpellSchoolCard.test.ts`
- `tests/components/source/SourceCard.test.ts`

---

## 🎯 Success Criteria

### Functional Requirements
- ✅ All 16 entity types supported in composable
- ✅ Kebab-case → snake_case conversion works transparently
- ✅ Reference cards show background images (256px, 10% opacity)
- ✅ Hover effects work (20% opacity on hover)
- ✅ Missing images handled gracefully

### Quality Requirements
- ✅ All new tests pass (100% pass rate)
- ✅ No regressions in existing tests (645 tests still passing)
- ✅ TypeScript compiles with no new errors
- ✅ ESLint passes with no warnings

### Verification Requirements
- ✅ Browser verification: All 10 reference list pages HTTP 200
- ✅ Visual verification: Background images visible on cards
- ✅ Manual testing: Hover effects work as expected
- ✅ Performance: No noticeable slowdown

---

## 🚀 Implementation Plan

### Phase 1: Composable Refactor (TDD)
**Estimated Time:** 30 minutes

1. Write tests for new `getImagePath` signature
2. Implement entity mapping logic
3. Test all 16 entity types
4. Test kebab-case → snake_case conversion
5. Verify validation logic

**Deliverable:** Refactored composable with 35 passing tests

---

### Phase 2: Update Main Entities
**Estimated Time:** 45 minutes

1. Update 6 main entity cards to new signature
2. Update 6 main entity detail pages to new signature
3. Verify all tests still pass
4. Browser verification

**Deliverable:** 12 files migrated, no regressions

---

### Phase 3: Update Reference Entities
**Estimated Time:** 90 minutes

1. Update 10 reference entity cards (background images)
2. Write 2 tests per card (20 total)
3. Verify all tests pass
4. Browser verification

**Deliverable:** 10 cards enhanced, 20 new tests passing

---

### Phase 4: Documentation & Commit
**Estimated Time:** 15 minutes

1. Update CHANGELOG.md
2. Update CURRENT_STATUS.md
3. Create handover document
4. Commit all changes

**Deliverable:** Complete documentation

---

**Total Estimated Time:** ~3 hours

---

## 📚 Future Enhancements

### Potential Improvements
1. **Detail Pages for Reference Entities:** Add hero images when detail pages created
2. **Image Fallbacks:** Generate default placeholder images for missing entities
3. **Image Preloading:** Preload hero images on list card hover
4. **Multiple Providers:** Support switching providers per entity type
5. **Image Gallery:** Modal to view full-size images

### Scalability
- Adding new entity types: Update union type + mapping (2 lines)
- Adding new providers: Update runtime config only
- Adding new sizes: Update ImageSize type + path logic

---

## 🎓 Design Principles Applied

### Type Safety
- Union types prevent invalid entity types at compile time
- No runtime errors from typos

### Developer Experience
- Developers use familiar route names (`'spell-schools'`)
- No need to remember folder naming conventions
- Auto-complete shows all available entity types

### Maintainability
- Single source of truth for entity → folder mapping
- Centralized validation logic
- Easy to extend (just update the map)

### Performance
- Simple object lookup (O(1) complexity)
- No regex, no string manipulation overhead
- Graceful degradation (no expensive error handling)

---

## 📖 Related Documentation

**Design Documents:**
- `docs/plans/2025-11-22-entity-images-design.md` - Original feature design (main entities)
- `docs/plans/2025-11-22-entity-images-expansion-design.md` - Expansion to all main entities

**Implementation Plans:**
- `docs/plans/2025-11-22-entity-images-implementation.md` - Races implementation
- `docs/plans/2025-11-22-entity-images-expansion-implementation.md` - All main entities

**Handover Documents:**
- `docs/HANDOVER-2025-11-22-ENTITY-IMAGES.md` - Races implementation
- `docs/HANDOVER-2025-11-22-ENTITY-IMAGES-EXPANSION.md` - All main entities

---

## ✅ Design Approved

**Next Steps:**
1. Create implementation plan with detailed tasks
2. Set up git worktree for isolated development
3. Execute implementation following TDD methodology

---

**Document Version:** 1.0
**Author:** Claude Code
**Approved By:** User
**Date:** 2025-11-22
