# Reference Entity Images Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Extend entity images to all 16 entity types (6 main + 10 reference) by refactoring the `useEntityImage` composable and adding background images to reference entity list cards.

**Architecture:** Refactor composable to return object with `getImagePath()` method that accepts all entity types and handles kebab-case → snake_case conversion transparently. Update main entity components to new signature, then add background images to reference entity cards following established pattern.

**Tech Stack:** Vue 3 Composition API, TypeScript, Vitest, NuxtImg, Docker

---

## Prerequisites

**Required Reading:**
- Design doc: `docs/plans/2025-11-22-reference-entity-images-design.md`
- TDD mandate: `CLAUDE.md` (TDD section is non-negotiable)
- Entity images pattern: `docs/HANDOVER-2025-11-22-ENTITY-IMAGES-EXPANSION.md`

**Verify Docker Environment:**
```bash
docker compose ps
# Expected: nuxt container running
```

**Test Baseline:**
```bash
docker compose exec nuxt npm run test
# Expected: 645 tests passing
```

---

## Phase 1: Refactor `useEntityImage` Composable (TDD)

### Task 1: Write Tests for New Signature

**Files:**
- Modify: `tests/composables/useEntityImage.test.ts`

**Step 1: Read existing test file**

```bash
cat tests/composables/useEntityImage.test.ts
```

**Step 2: Write failing tests for new signature**

Add to `tests/composables/useEntityImage.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { useEntityImage } from '~/composables/useEntityImage'

describe('useEntityImage', () => {
  describe('getImagePath', () => {
    const { getImagePath } = useEntityImage()

    // Main entities (6 tests)
    it('returns correct path for races', () => {
      const result = getImagePath('races', 'dragonborn', 256)
      expect(result).toBe('/images/generated/conversions/256/races/stability-ai/dragonborn.png')
    })

    it('returns correct path for classes', () => {
      const result = getImagePath('classes', 'wizard', 256)
      expect(result).toBe('/images/generated/conversions/256/classes/stability-ai/wizard.png')
    })

    it('returns correct path for backgrounds', () => {
      const result = getImagePath('backgrounds', 'acolyte', 256)
      expect(result).toBe('/images/generated/conversions/256/backgrounds/stability-ai/acolyte.png')
    })

    it('returns correct path for feats', () => {
      const result = getImagePath('feats', 'alert', 256)
      expect(result).toBe('/images/generated/conversions/256/feats/stability-ai/alert.png')
    })

    it('returns correct path for spells', () => {
      const result = getImagePath('spells', 'fireball', 256)
      expect(result).toBe('/images/generated/conversions/256/spells/stability-ai/fireball.png')
    })

    it('returns correct path for items', () => {
      const result = getImagePath('items', 'longsword', 256)
      expect(result).toBe('/images/generated/conversions/256/items/stability-ai/longsword.png')
    })

    // Reference entities with conversion (5 tests)
    it('converts ability-scores to ability_scores', () => {
      const result = getImagePath('ability-scores', 'strength', 256)
      expect(result).toBe('/images/generated/conversions/256/ability_scores/stability-ai/strength.png')
    })

    it('converts spell-schools to spell_schools', () => {
      const result = getImagePath('spell-schools', 'evocation', 256)
      expect(result).toBe('/images/generated/conversions/256/spell_schools/stability-ai/evocation.png')
    })

    it('converts item-types to item_types', () => {
      const result = getImagePath('item-types', 'weapon', 256)
      expect(result).toBe('/images/generated/conversions/256/item_types/stability-ai/weapon.png')
    })

    it('converts proficiency-types to proficiency_types', () => {
      const result = getImagePath('proficiency-types', 'armor', 256)
      expect(result).toBe('/images/generated/conversions/256/proficiency_types/stability-ai/armor.png')
    })

    it('converts damage-types to damage_types', () => {
      const result = getImagePath('damage-types', 'fire', 256)
      expect(result).toBe('/images/generated/conversions/256/damage_types/stability-ai/fire.png')
    })

    // Reference entities with direct match (5 tests)
    it('uses direct match for conditions', () => {
      const result = getImagePath('conditions', 'blinded', 256)
      expect(result).toBe('/images/generated/conversions/256/conditions/stability-ai/blinded.png')
    })

    it('uses direct match for languages', () => {
      const result = getImagePath('languages', 'common', 256)
      expect(result).toBe('/images/generated/conversions/256/languages/stability-ai/common.png')
    })

    it('uses direct match for sizes', () => {
      const result = getImagePath('sizes', 'medium', 256)
      expect(result).toBe('/images/generated/conversions/256/sizes/stability-ai/medium.png')
    })

    it('uses direct match for skills', () => {
      const result = getImagePath('skills', 'acrobatics', 256)
      expect(result).toBe('/images/generated/conversions/256/skills/stability-ai/acrobatics.png')
    })

    it('uses direct match for sources', () => {
      const result = getImagePath('sources', 'phb', 256)
      expect(result).toBe('/images/generated/conversions/256/sources/stability-ai/phb.png')
    })

    // Size variants (3 tests)
    it('returns 256px conversion path by default', () => {
      const result = getImagePath('skills', 'acrobatics')
      expect(result).toBe('/images/generated/conversions/256/skills/stability-ai/acrobatics.png')
    })

    it('returns 512px conversion path when specified', () => {
      const result = getImagePath('skills', 'acrobatics', 512)
      expect(result).toBe('/images/generated/conversions/512/skills/stability-ai/acrobatics.png')
    })

    it('returns original path when specified', () => {
      const result = getImagePath('skills', 'acrobatics', 'original')
      expect(result).toBe('/images/generated/skills/stability-ai/acrobatics.png')
    })

    // Validation (3 tests)
    it('returns null for empty slug', () => {
      const result = getImagePath('skills', '', 256)
      expect(result).toBeNull()
    })

    it('returns null for whitespace-only slug', () => {
      const result = getImagePath('skills', '   ', 256)
      expect(result).toBeNull()
    })

    it('handles slug with hyphens correctly', () => {
      const result = getImagePath('skills', 'animal-handling', 256)
      expect(result).toBe('/images/generated/conversions/256/skills/stability-ai/animal-handling.png')
    })
  })
})
```

**Step 3: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- useEntityImage.test.ts
```

Expected: All new tests FAIL (composable doesn't have `getImagePath` yet)

**Step 4: Commit**

```bash
git add tests/composables/useEntityImage.test.ts
git commit -m "test: Add tests for refactored useEntityImage composable

- Add 25 tests for getImagePath method
- Test all 16 entity types (6 main + 10 reference)
- Test kebab-case to snake_case conversion
- Test size variants and validation

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 2: Implement Refactored Composable

**Files:**
- Modify: `app/composables/useEntityImage.ts`

**Step 1: Read existing composable**

```bash
cat app/composables/useEntityImage.ts
```

**Step 2: Refactor to new signature**

Replace entire contents of `app/composables/useEntityImage.ts`:

```typescript
/**
 * Entity image composable
 * Provides image paths for all entity types (main + reference)
 */

export type EntityType =
  // Main entities
  | 'races' | 'classes' | 'backgrounds' | 'feats' | 'spells' | 'items'
  // Reference entities
  | 'ability-scores' | 'conditions' | 'damage-types' | 'item-types'
  | 'languages' | 'proficiency-types' | 'sizes' | 'skills'
  | 'spell-schools' | 'sources'

export type ImageSize = '256' | '512' | 'original'

/**
 * Entity type to folder name mapping
 * Handles kebab-case (frontend routes) → snake_case (image folders)
 */
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

/**
 * Get image path for entity
 */
export function useEntityImage() {
  const config = useRuntimeConfig()
  const provider = config.public.imageProvider || 'stability-ai'

  return {
    /**
     * Generate image path for entity
     * @param entityType - Entity type (e.g., 'skills', 'spell-schools')
     * @param slug - Entity slug (e.g., 'acrobatics', 'fireball')
     * @param size - Image size variant (default: 256)
     * @returns Image path or null if invalid
     */
    getImagePath(
      entityType: EntityType,
      slug: string,
      size: ImageSize = '256'
    ): string | null {
      // Validate slug
      if (!slug || slug.trim() === '') {
        return null
      }

      // Map entity type to folder name
      const folderName = ENTITY_FOLDER_MAP[entityType]
      if (!folderName) {
        return null
      }

      // Build path based on size
      if (size === 'original') {
        return `/images/generated/${folderName}/${provider}/${slug}.png`
      }

      return `/images/generated/conversions/${size}/${folderName}/${provider}/${slug}.png`
    }
  }
}
```

**Step 3: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- useEntityImage.test.ts
```

Expected: All 25 tests PASS ✅

**Step 4: Run full test suite to check for regressions**

```bash
docker compose exec nuxt npm run test
```

Expected: Some tests will fail (existing code uses old signature)

**Step 5: Commit**

```bash
git add app/composables/useEntityImage.ts
git commit -m "refactor: Update useEntityImage to support all 16 entity types

- Return object with getImagePath method
- Support 6 main + 10 reference entity types
- Handle kebab-case to snake_case conversion
- Maintain validation logic

Breaking change: Signature changed from function to object
Migration required for existing usage

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase 2: Update Main Entity Cards (6 files)

### Task 3: Update RaceCard

**Files:**
- Modify: `app/components/race/RaceCard.vue:52-54`

**Step 1: Read existing RaceCard**

```bash
cat app/components/race/RaceCard.vue | head -60
```

**Step 2: Update to new composable signature**

Find lines 52-54 (approximately):
```vue
const { getImagePath } = useEntityImage()
const backgroundImageUrl = computed(() => getImagePath('races', props.race.slug, 256))
```

Already using new signature! No changes needed.

**Step 3: Verify tests still pass**

```bash
docker compose exec nuxt npm run test -- RaceCard.test.ts
```

Expected: All tests PASS ✅ (already using new signature)

---

### Task 4: Update ClassCard

**Files:**
- Modify: `app/components/class/ClassCard.vue`

**Step 1: Find composable usage**

```bash
grep -n "useEntityImage" app/components/class/ClassCard.vue
```

**Step 2: Update to new signature**

Replace:
```typescript
const backgroundImageUrl = computed(() =>
  useEntityImage(props.class.slug, 'classes', 256)
)
```

With:
```typescript
const { getImagePath } = useEntityImage()
const backgroundImageUrl = computed(() =>
  getImagePath('classes', props.class.slug, 256)
)
```

**Step 3: Run tests**

```bash
docker compose exec nuxt npm run test -- ClassCard.test.ts
```

Expected: All tests PASS ✅

**Step 4: Commit**

```bash
git add app/components/class/ClassCard.vue
git commit -m "refactor: Update ClassCard to new useEntityImage signature

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 5: Update BackgroundCard

**Files:**
- Modify: `app/components/background/BackgroundCard.vue`

**Step 1: Update to new signature**

Replace:
```typescript
const backgroundImageUrl = computed(() =>
  useEntityImage(props.background.slug, 'backgrounds', 256)
)
```

With:
```typescript
const { getImagePath } = useEntityImage()
const backgroundImageUrl = computed(() =>
  getImagePath('backgrounds', props.background.slug, 256)
)
```

**Step 2: Run tests**

```bash
docker compose exec nuxt npm run test -- BackgroundCard.test.ts
```

Expected: All tests PASS ✅

**Step 3: Commit**

```bash
git add app/components/background/BackgroundCard.vue
git commit -m "refactor: Update BackgroundCard to new useEntityImage signature

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 6: Update FeatCard

**Files:**
- Modify: `app/components/feat/FeatCard.vue`

**Step 1: Update to new signature**

Replace:
```typescript
const backgroundImageUrl = computed(() =>
  useEntityImage(props.feat.slug, 'feats', 256)
)
```

With:
```typescript
const { getImagePath } = useEntityImage()
const backgroundImageUrl = computed(() =>
  getImagePath('feats', props.feat.slug, 256)
)
```

**Step 2: Run tests**

```bash
docker compose exec nuxt npm run test -- FeatCard.test.ts
```

Expected: All tests PASS ✅

**Step 3: Commit**

```bash
git add app/components/feat/FeatCard.vue
git commit -m "refactor: Update FeatCard to new useEntityImage signature

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 7: Update SpellCard

**Files:**
- Modify: `app/components/spell/SpellCard.vue`

**Step 1: Update to new signature**

Replace:
```typescript
const backgroundImageUrl = computed(() =>
  useEntityImage(props.spell.slug, 'spells', 256)
)
```

With:
```typescript
const { getImagePath } = useEntityImage()
const backgroundImageUrl = computed(() =>
  getImagePath('spells', props.spell.slug, 256)
)
```

**Step 2: Run tests**

```bash
docker compose exec nuxt npm run test -- SpellCard.test.ts
```

Expected: All tests PASS ✅

**Step 3: Commit**

```bash
git add app/components/spell/SpellCard.vue
git commit -m "refactor: Update SpellCard to new useEntityImage signature

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 8: Update ItemCard

**Files:**
- Modify: `app/components/item/ItemCard.vue`

**Step 1: Update to new signature**

Replace:
```typescript
const backgroundImageUrl = computed(() =>
  useEntityImage(props.item.slug, 'items', 256)
)
```

With:
```typescript
const { getImagePath } = useEntityImage()
const backgroundImageUrl = computed(() =>
  getImagePath('items', props.item.slug, 256)
)
```

**Step 2: Run tests**

```bash
docker compose exec nuxt npm run test -- ItemCard.test.ts
```

Expected: All tests PASS ✅

**Step 3: Commit**

```bash
git add app/components/item/ItemCard.vue
git commit -m "refactor: Update ItemCard to new useEntityImage signature

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase 3: Update Main Entity Detail Pages (6 files)

### Task 9: Update Race Detail Page

**Files:**
- Modify: `app/pages/races/[slug].vue`

**Step 1: Find composable usage**

```bash
grep -n "useEntityImage" app/pages/races/[slug].vue
```

**Step 2: Update to new signature**

Replace:
```typescript
const entityImageUrl = computed(() =>
  entity.value ? useEntityImage(entity.value.slug, 'races', 512) : null
)
```

With:
```typescript
const { getImagePath } = useEntityImage()
const entityImageUrl = computed(() =>
  entity.value ? getImagePath('races', entity.value.slug, 512) : null
)
```

**Step 3: Verify in browser**

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/races/dragonborn
```

Expected: 200

**Step 4: Commit**

```bash
git add app/pages/races/[slug].vue
git commit -m "refactor: Update race detail page to new useEntityImage signature

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 10: Update Remaining Detail Pages (Batch)

**Files:**
- Modify: `app/pages/classes/[slug].vue`
- Modify: `app/pages/backgrounds/[slug].vue`
- Modify: `app/pages/feats/[slug].vue`
- Modify: `app/pages/spells/[slug].vue`
- Modify: `app/pages/items/[slug].vue`

**Step 1: Update all 5 detail pages**

For each file, replace:
```typescript
const entityImageUrl = computed(() =>
  entity.value ? useEntityImage(entity.value.slug, 'ENTITY_TYPE', 512) : null
)
```

With:
```typescript
const { getImagePath } = useEntityImage()
const entityImageUrl = computed(() =>
  entity.value ? getImagePath('ENTITY_TYPE', entity.value.slug, 512) : null
)
```

Where `ENTITY_TYPE` is: `classes`, `backgrounds`, `feats`, `spells`, `items`

**Step 2: Verify all pages work**

```bash
for entity in classes backgrounds feats spells items; do
  echo -n "$entity: "
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/$entity
done
```

Expected: All return 200

**Step 3: Run full test suite**

```bash
docker compose exec nuxt npm run test
```

Expected: All 645 tests PASS ✅ (main entities migrated successfully)

**Step 4: Commit**

```bash
git add app/pages/classes/[slug].vue app/pages/backgrounds/[slug].vue app/pages/feats/[slug].vue app/pages/spells/[slug].vue app/pages/items/[slug].vue
git commit -m "refactor: Update all detail pages to new useEntityImage signature

- Classes, backgrounds, feats, spells, items
- All main entity migrations complete
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Phase 4: Add Background Images to Reference Cards (10 files)

### Task 11: Update SkillCard (TDD)

**Files:**
- Modify: `app/components/skill/SkillCard.vue`
- Modify: `tests/components/skill/SkillCard.test.ts`

**Step 1: Write failing tests**

Add to `tests/components/skill/SkillCard.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SkillCard from '~/components/skill/SkillCard.vue'

describe('SkillCard', () => {
  const mockSkill = {
    id: 1,
    slug: 'acrobatics',
    name: 'Acrobatics',
    ability_score: {
      id: 2,
      code: 'DEX',
      name: 'Dexterity'
    }
  }

  // Existing tests...

  describe('background images', () => {
    it('computes background image URL correctly', async () => {
      const wrapper = await mountSuspended(SkillCard, {
        props: { skill: mockSkill }
      })

      const url = wrapper.vm.backgroundImageUrl
      expect(url).toBe('/images/generated/conversions/256/skills/stability-ai/acrobatics.png')
    })

    it('applies background image styles when URL exists', async () => {
      const wrapper = await mountSuspended(SkillCard, {
        props: { skill: mockSkill }
      })

      const card = wrapper.find('.group')
      const style = card.attributes('style')
      expect(style).toContain('background-image')
      expect(style).toContain('acrobatics.png')
    })
  })
})
```

**Step 2: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- SkillCard.test.ts
```

Expected: New tests FAIL (no backgroundImageUrl computed property yet)

**Step 3: Update SkillCard component**

Replace entire `app/components/skill/SkillCard.vue`:

```vue
<script setup lang="ts">
interface Skill {
  id: number
  slug: string
  name: string
  ability_score?: {
    id: number
    code: string
    name: string
  } | null
}

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
      <!-- Skill Name -->
      <h3 class="text-xl font-semibold text-gray-900 dark:text-gray-100">
        {{ skill.name }}
      </h3>

      <!-- Ability Score (if present) -->
      <div
        v-if="skill.ability_score"
        class="space-y-2"
      >
        <div class="flex items-center gap-2">
          <UBadge
            color="info"
            variant="solid"
            size="md"
          >
            {{ skill.ability_score.code }}
          </UBadge>
        </div>
        <p class="text-sm text-gray-600 dark:text-gray-400">
          {{ skill.ability_score.name }}
        </p>
      </div>

      <!-- Category Badge -->
      <div class="flex items-center gap-2">
        <UBadge
          color="neutral"
          variant="soft"
          size="xs"
        >
          Skill
        </UBadge>
      </div>
    </div>
  </div>
</template>
```

**Step 4: Run tests to verify they pass**

```bash
docker compose exec nuxt npm run test -- SkillCard.test.ts
```

Expected: All tests PASS ✅

**Step 5: Verify in browser**

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/skills
```

Expected: 200

**Step 6: Commit**

```bash
git add app/components/skill/SkillCard.vue tests/components/skill/SkillCard.test.ts
git commit -m "feat: Add background images to SkillCard (TDD)

- Add background image with 256px variant
- 10% opacity (90% overlay), 20% on hover
- Add 2 tests for image functionality
- All tests passing

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 12: Update SpellSchoolCard (TDD)

**Files:**
- Modify: `app/components/spell-school/SpellSchoolCard.vue`
- Modify: `tests/components/spell-school/SpellSchoolCard.test.ts`

**Step 1: Write failing tests**

Add to `tests/components/spell-school/SpellSchoolCard.test.ts`:

```typescript
describe('background images', () => {
  it('computes background image URL correctly', async () => {
    const wrapper = await mountSuspended(SpellSchoolCard, {
      props: {
        spellSchool: {
          id: 1,
          slug: 'evocation',
          code: 'EVO',
          name: 'Evocation',
          description: 'Evocation spells'
        }
      }
    })

    const url = wrapper.vm.backgroundImageUrl
    expect(url).toBe('/images/generated/conversions/256/spell_schools/stability-ai/evocation.png')
  })

  it('applies background image styles when URL exists', async () => {
    const wrapper = await mountSuspended(SpellSchoolCard, {
      props: {
        spellSchool: {
          id: 1,
          slug: 'evocation',
          code: 'EVO',
          name: 'Evocation'
        }
      }
    })

    const card = wrapper.find('.group')
    const style = card.attributes('style')
    expect(style).toContain('background-image')
  })
})
```

**Step 2: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test -- SpellSchoolCard.test.ts
```

Expected: FAIL

**Step 3: Update component (following SkillCard pattern)**

Update `app/components/spell-school/SpellSchoolCard.vue` to add:
- `const { getImagePath } = useEntityImage()`
- `const backgroundImageUrl = computed(() => getImagePath('spell-schools', props.spellSchool.slug, 256))`
- Change `<UCard>` to `<div>` with background image styles
- Add overlay classes (same as SkillCard)

**Step 4: Run tests**

```bash
docker compose exec nuxt npm run test -- SpellSchoolCard.test.ts
```

Expected: All tests PASS ✅

**Step 5: Commit**

```bash
git add app/components/spell-school/SpellSchoolCard.vue tests/components/spell-school/SpellSchoolCard.test.ts
git commit -m "feat: Add background images to SpellSchoolCard (TDD)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 13: Update Remaining Reference Cards (Batch, TDD)

**Files:**
- `app/components/ability-score/AbilityScoreCard.vue` + test
- `app/components/condition/ConditionCard.vue` + test
- `app/components/damage-type/DamageTypeCard.vue` + test
- `app/components/item-type/ItemTypeCard.vue` + test
- `app/components/language/LanguageCard.vue` + test
- `app/components/proficiency-type/ProficiencyTypeCard.vue` + test
- `app/components/size/SizeCard.vue` + test
- `app/components/source/SourceCard.vue` + test

**Entity Type Mapping:**
- AbilityScoreCard → `'ability-scores'`
- ConditionCard → `'conditions'`
- DamageTypeCard → `'damage-types'`
- ItemTypeCard → `'item-types'`
- LanguageCard → `'languages'`
- ProficiencyTypeCard → `'proficiency-types'`
- SizeCard → `'sizes'`
- SourceCard → `'sources'`

**Step 1: For each card, write tests first**

Add 2 tests to each card's test file:
```typescript
describe('background images', () => {
  it('computes background image URL correctly', () => { ... })
  it('applies background image styles when URL exists', () => { ... })
})
```

**Step 2: Run tests to verify they fail**

```bash
docker compose exec nuxt npm run test
```

Expected: New tests FAIL

**Step 3: Update each card component**

For each card:
1. Add `const { getImagePath } = useEntityImage()`
2. Add `const backgroundImageUrl = computed(() => getImagePath('ENTITY_TYPE', props.ENTITY.slug, 256))`
3. Replace `<UCard>` with `<div>` + background image pattern
4. Keep all existing content inside `<div class="relative z-10 p-4 space-y-3">`

**Step 4: Run tests after each card**

```bash
docker compose exec nuxt npm run test -- AbilityScoreCard.test.ts
# ... repeat for each card
```

**Step 5: Commit each card individually**

```bash
git add app/components/ability-score/AbilityScoreCard.vue tests/components/ability-score/AbilityScoreCard.test.ts
git commit -m "feat: Add background images to AbilityScoreCard (TDD)"

# Repeat for each card
```

---

### Task 14: Full Test Suite Verification

**Step 1: Run complete test suite**

```bash
docker compose exec nuxt npm run test
```

Expected: ~680 tests PASS ✅ (645 original + ~35 new)

**Step 2: Verify all reference list pages**

```bash
for entity in ability-scores conditions damage-types item-types languages proficiency-types sizes skills spell-schools sources; do
  echo -n "$entity: "
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3000/$entity
done
```

Expected: All return 200

**Step 3: TypeScript check**

```bash
docker compose exec nuxt npm run typecheck
```

Expected: Same errors as before (13 pre-existing)

**Step 4: ESLint check**

```bash
docker compose exec nuxt npm run lint
```

Expected: 0 errors

---

## Phase 5: Documentation & Completion

### Task 15: Update CHANGELOG

**Files:**
- Modify: `CHANGELOG.md`

**Step 1: Add entry to Unreleased section**

```markdown
## [Unreleased]

### Added
- Entity images expanded to all 16 entity types (2025-11-22)
  - Refactored `useEntityImage` composable to support main + reference entities
  - Added background images to 10 reference entity cards
  - Handles kebab-case to snake_case conversion transparently
  - All 680 tests passing
```

**Step 2: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG for reference entity images

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 16: Update CURRENT_STATUS

**Files:**
- Modify: `docs/CURRENT_STATUS.md`

**Step 1: Update header stats**

Line 7: Update test count:
```markdown
**Test Coverage:** 680/680 tests passing (100% pass rate!) ✨
```

**Step 2: Update Entity Images section**

Around line 52, update:
```markdown
### Entity Images Feature
**Status:** ✅ Complete for All 16 Entity Types (Main + Reference)

**All Entity Pages:**
- **Main Entities (6):** Races, Classes, Backgrounds, Feats, Spells, Items
  - Detail Pages: CV-style hero images (512px, right-aligned)
  - List Cards: Background images (256px, 10% opacity, 20% on hover)
- **Reference Entities (10):** Skills, Spell Schools, Ability Scores, Conditions, Damage Types, Item Types, Languages, Proficiency Types, Sizes, Sources
  - List Cards: Background images (256px, 10% opacity, 20% on hover)
```

**Step 3: Update test count breakdown**

Around line 274:
```markdown
**Test Coverage:**
- ✅ **680 tests total** (ALL PASSING ✅) ⭐ (+35 reference entity images)
```

**Step 4: Commit**

```bash
git add docs/CURRENT_STATUS.md
git commit -m "docs: Update CURRENT_STATUS for reference entity images

- Test count: 680 (was 645)
- Entity images: All 16 entity types
- Reference entities: 10 cards with background images

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

### Task 17: Create Handover Document

**Files:**
- Create: `docs/HANDOVER-2025-11-22-REFERENCE-ENTITY-IMAGES.md`

**Step 1: Create handover document**

```markdown
# Handover Document: Reference Entity Images

**Date:** 2025-11-22
**Session Focus:** Extend entity images to all 16 entity types (6 main + 10 reference)
**Status:** ✅ **COMPLETE**
**Test Results:** 680/680 tests passing (100% pass rate)

---

## 📋 Executive Summary

Successfully extended the entity images feature to support all entity types by refactoring the `useEntityImage` composable and adding background images to 10 reference entity list cards.

**Key Achievements:**
- ✅ Composable supports all 16 entity types (6 main + 10 reference)
- ✅ Handles kebab-case → snake_case conversion transparently
- ✅ 10 reference entity cards enhanced with background images
- ✅ 24 files modified, ~35 new tests added
- ✅ All 680 tests passing (100% pass rate)
- ✅ Zero regressions, backward compatible

---

## 🎯 What Was Built

### Composable Refactor
- New signature: `useEntityImage().getImagePath(entityType, slug, size)`
- Type-safe: Union type for all 16 entity types
- Smart mapping: Converts `'spell-schools'` → `'spell_schools'`

### Reference Cards Enhanced (10 files)
1. SkillCard
2. SpellSchoolCard
3. AbilityScoreCard
4. ConditionCard
5. DamageTypeCard
6. ItemTypeCard
7. LanguageCard
8. ProficiencyTypeCard
9. SizeCard
10. SourceCard

### Main Entities Migrated (12 files)
- 6 card components updated to new signature
- 6 detail pages updated to new signature

---

## 🧪 Test Results

**Total Tests:** 680/680 passing (100% pass rate)
**New Tests:** 35 (25 composable + 10 card tests)
**Duration:** ~3 hours

---

## 📝 Documentation

- Design: `docs/plans/2025-11-22-reference-entity-images-design.md`
- Implementation: `docs/plans/2025-11-22-reference-entity-images-implementation.md`
- CHANGELOG: Updated
- CURRENT_STATUS: Updated

---

**Status:** Ready for production! 🚀
```

**Step 2: Commit**

```bash
git add docs/HANDOVER-2025-11-22-REFERENCE-ENTITY-IMAGES.md
git commit -m "docs: Add handover document for reference entity images

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## 🎯 Final Verification

### Visual Verification Checklist

**Reference List Pages (10 pages):**
```bash
open http://localhost:3000/skills
open http://localhost:3000/spell-schools
open http://localhost:3000/ability-scores
open http://localhost:3000/conditions
open http://localhost:3000/damage-types
open http://localhost:3000/item-types
open http://localhost:3000/languages
open http://localhost:3000/proficiency-types
open http://localhost:3000/sizes
open http://localhost:3000/sources
```

**Verify:**
- ✅ Background images visible (subtle, 10% opacity)
- ✅ Hover effects work (20% opacity)
- ✅ Cards still readable
- ✅ No console errors

**Main Entity Pages (6 pages):**
```bash
open http://localhost:3000/races
open http://localhost:3000/classes
open http://localhost:3000/backgrounds
open http://localhost:3000/feats
open http://localhost:3000/spells
open http://localhost:3000/items
```

**Verify:**
- ✅ Background images still work
- ✅ No regressions

---

## 📊 Success Metrics

**Code Quality:**
- ✅ TypeScript: 13 errors (unchanged, pre-existing)
- ✅ ESLint: 0 errors
- ✅ Tests: 680/680 passing (100%)

**Coverage:**
- ✅ All 16 entity types supported
- ✅ All 10 reference cards enhanced
- ✅ All 12 main entity files migrated

**Performance:**
- ✅ No noticeable slowdown
- ✅ Lazy loading via NuxtImg
- ✅ Graceful degradation for missing images

---

## 🎓 Pattern Reference

**Reference Card Background Image Pattern:**

```vue
<script setup lang="ts">
const { getImagePath } = useEntityImage()
const backgroundImageUrl = computed(() =>
  getImagePath('ENTITY-TYPE', props.entity.slug, 256)
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
      <!-- content -->
    </div>
  </div>
</template>
```

**Entity Type Reference:**
- Main: `'races'` `'classes'` `'backgrounds'` `'feats'` `'spells'` `'items'`
- Reference: `'ability-scores'` `'conditions'` `'damage-types'` `'item-types'` `'languages'` `'proficiency-types'` `'sizes'` `'skills'` `'spell-schools'` `'sources'`

---

**Plan Complete!** 🎉

**Estimated Time:** ~3 hours
**Complexity:** Medium (refactoring + migration + enhancement)
**Risk:** Low (TDD, incremental commits, backward compatible)
