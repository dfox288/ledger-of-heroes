# Backgrounds Quick Stats Panel Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add 2/3 quick stats + 1/3 image grid layout to backgrounds detail page, showing actual proficiency/language/equipment data.

**Architecture:** Extract BackgroundCard's proficiency-parsing logic into a reusable `useBackgroundStats` composable, then use it in both the card and detail page. Replace full-width image with grid layout matching classes/races pattern.

**Tech Stack:** Nuxt 4, Vue 3 Composition API, TypeScript, Vitest, Docker

---

## Prerequisites

- Backend running: `cd ../importer && docker compose up -d`
- Frontend Docker running: `docker compose up -d`
- Clean test baseline: All tests passing
- Feature branch: `feature/backgrounds-quick-stats`

---

## Task 1: Create `useBackgroundStats` Composable (TDD)

**Files:**
- Create: `app/composables/useBackgroundStats.ts`
- Create: `tests/composables/useBackgroundStats.test.ts`

### Step 1: Write the failing tests

**File:** `tests/composables/useBackgroundStats.test.ts`

```typescript
import { describe, it, expect } from 'vitest'
import { ref } from 'vue'
import { useBackgroundStats } from '~/composables/useBackgroundStats'
import type { Background } from '~/types/api/entities'
import { ITEM_ID_GOLD_PIECE } from '~/constants/items'

describe('useBackgroundStats', () => {
  it('extracts skill proficiency names', () => {
    const background = ref<Background>({
      id: 1,
      slug: 'charlatan',
      name: 'Charlatan',
      proficiencies: [
        { proficiency_type: 'skill', skill: { id: 1, name: 'Deception', slug: 'deception' } },
        { proficiency_type: 'skill', skill: { id: 2, name: 'Sleight of Hand', slug: 'sleight-of-hand' } }
      ]
    } as Background)

    const { skillProficiencies } = useBackgroundStats(background)

    expect(skillProficiencies.value).toEqual(['Deception', 'Sleight of Hand'])
  })

  it('extracts tool proficiency names', () => {
    const background = ref<Background>({
      id: 1,
      slug: 'charlatan',
      name: 'Charlatan',
      proficiencies: [
        { proficiency_type: 'tool', tool: { id: 1, name: 'Disguise Kit' } },
        { proficiency_type: 'tool', tool: { id: 2, name: 'Forgery Kit' } }
      ]
    } as Background)

    const { toolProficiencies } = useBackgroundStats(background)

    expect(toolProficiencies.value).toEqual(['Disguise Kit', 'Forgery Kit'])
  })

  it('extracts language names', () => {
    const background = ref<Background>({
      id: 1,
      slug: 'acolyte',
      name: 'Acolyte',
      languages: [
        { id: 1, name: 'Common', slug: 'common' },
        { id: 2, name: 'Celestial', slug: 'celestial' }
      ]
    } as Background)

    const { languages } = useBackgroundStats(background)

    expect(languages.value).toEqual(['Common', 'Celestial'])
  })

  it('counts equipment items excluding gold', () => {
    const background = ref<Background>({
      id: 1,
      slug: 'acolyte',
      name: 'Acolyte',
      equipment: [
        { item_id: 100, item_name: 'Holy Symbol', quantity: 1 },
        { item_id: 101, item_name: 'Prayer Book', quantity: 1 },
        { item_id: 102, item_name: 'Incense', quantity: 5 },
        { item_id: ITEM_ID_GOLD_PIECE, item_name: 'Gold Piece', quantity: 15 }
      ]
    } as Background)

    const { equipmentCount } = useBackgroundStats(background)

    expect(equipmentCount.value).toBe(3)
  })

  it('finds starting gold from equipment array', () => {
    const background = ref<Background>({
      id: 1,
      slug: 'charlatan',
      name: 'Charlatan',
      equipment: [
        { item_id: 100, item_name: 'Fine Clothes', quantity: 1 },
        { item_id: ITEM_ID_GOLD_PIECE, item_name: 'Gold Piece', quantity: 15 }
      ]
    } as Background)

    const { startingGold } = useBackgroundStats(background)

    expect(startingGold.value).toBe(15)
  })

  it('handles missing proficiencies gracefully', () => {
    const background = ref<Background>({
      id: 1,
      slug: 'test',
      name: 'Test'
    } as Background)

    const { skillProficiencies, toolProficiencies } = useBackgroundStats(background)

    expect(skillProficiencies.value).toEqual([])
    expect(toolProficiencies.value).toEqual([])
  })

  it('handles missing languages gracefully', () => {
    const background = ref<Background>({
      id: 1,
      slug: 'test',
      name: 'Test'
    } as Background)

    const { languages } = useBackgroundStats(background)

    expect(languages.value).toEqual([])
  })

  it('handles missing equipment gracefully', () => {
    const background = ref<Background>({
      id: 1,
      slug: 'test',
      name: 'Test'
    } as Background)

    const { equipmentCount, startingGold } = useBackgroundStats(background)

    expect(equipmentCount.value).toBe(0)
    expect(startingGold.value).toBeNull()
  })

  it('handles null background', () => {
    const background = ref<Background | null>(null)

    const { skillProficiencies, toolProficiencies, languages, equipmentCount, startingGold } =
      useBackgroundStats(background)

    expect(skillProficiencies.value).toEqual([])
    expect(toolProficiencies.value).toEqual([])
    expect(languages.value).toEqual([])
    expect(equipmentCount.value).toBe(0)
    expect(startingGold.value).toBeNull()
  })

  it('filters out proficiencies without skill data', () => {
    const background = ref<Background>({
      id: 1,
      slug: 'test',
      name: 'Test',
      proficiencies: [
        { proficiency_type: 'skill', skill: { id: 1, name: 'Deception', slug: 'deception' } },
        { proficiency_type: 'skill', skill: null } as any
      ]
    } as Background)

    const { skillProficiencies } = useBackgroundStats(background)

    expect(skillProficiencies.value).toEqual(['Deception'])
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/composables/useBackgroundStats.test.ts
```

**Expected:** All tests FAIL with "Cannot find module '~/composables/useBackgroundStats'"

### Step 3: Write minimal implementation

**File:** `app/composables/useBackgroundStats.ts`

```typescript
import type { Ref } from 'vue'
import type { Background } from '~/types/api/entities'
import { ITEM_ID_GOLD_PIECE } from '~/constants/items'

/**
 * useBackgroundStats
 *
 * Extracts proficiency, language, and equipment data from a Background entity.
 * Used by both BackgroundCard and backgrounds detail page for consistency.
 *
 * @param background - Reactive Background entity (can be null during loading)
 * @returns Computed stats (skills, tools, languages, equipment, gold)
 *
 * @example
 * const { skillProficiencies, languages, startingGold } = useBackgroundStats(entity)
 */
export function useBackgroundStats(background: Ref<Background | null>) {
  /**
   * Extract skill proficiency names
   * Filters by proficiency_type='skill' and extracts skill.name
   */
  const skillProficiencies = computed(() => {
    if (!background.value?.proficiencies) return []

    return background.value.proficiencies
      .filter(p => p.proficiency_type === 'skill')
      .map(p => p.skill?.name)
      .filter(Boolean) as string[]
  })

  /**
   * Extract tool proficiency names
   * Filters by proficiency_type='tool' and extracts tool.name
   */
  const toolProficiencies = computed(() => {
    if (!background.value?.proficiencies) return []

    return background.value.proficiencies
      .filter(p => p.proficiency_type === 'tool')
      .map(p => p.tool?.name)
      .filter(Boolean) as string[]
  })

  /**
   * Extract language names
   */
  const languages = computed(() => {
    if (!background.value?.languages) return []

    return background.value.languages
      .map(l => l.name)
      .filter(Boolean) as string[]
  })

  /**
   * Count equipment items (excluding gold)
   */
  const equipmentCount = computed(() => {
    if (!background.value?.equipment) return 0

    return background.value.equipment.filter(
      eq => eq.item_id !== ITEM_ID_GOLD_PIECE
    ).length
  })

  /**
   * Extract starting gold quantity
   */
  const startingGold = computed(() => {
    if (!background.value?.equipment) return null

    const goldItem = background.value.equipment.find(
      eq => eq.item_id === ITEM_ID_GOLD_PIECE
    )

    return goldItem?.quantity || null
  })

  return {
    skillProficiencies,
    toolProficiencies,
    languages,
    equipmentCount,
    startingGold
  }
}
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/composables/useBackgroundStats.test.ts
```

**Expected:** All 10 tests PASS

### Step 5: Run full test suite

```bash
docker compose exec nuxt npm run test
```

**Expected:** All tests PASS (baseline + new composable tests)

### Step 6: Commit

```bash
git add app/composables/useBackgroundStats.ts tests/composables/useBackgroundStats.test.ts
git commit -m "feat: Create useBackgroundStats composable for proficiency parsing

- Extracts skill/tool proficiency names
- Extracts language names
- Counts equipment items (excluding gold)
- Finds starting gold from equipment
- Handles missing data gracefully
- 10 tests, all passing

Enables code reuse between BackgroundCard and detail page.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Update BackgroundCard to Use Composable

**Files:**
- Modify: `app/components/background/BackgroundCard.vue:1-200`
- Modify: `tests/components/background/BackgroundCard.test.ts` (if needed)

### Step 1: Write failing test for composable integration

**File:** `tests/components/background/BackgroundCard.test.ts`

Add this test if the file exists:

```typescript
it('uses useBackgroundStats composable', async () => {
  const background = {
    id: 1,
    slug: 'charlatan',
    name: 'Charlatan',
    proficiencies: [
      { proficiency_type: 'skill', skill: { id: 1, name: 'Deception', slug: 'deception' } },
      { proficiency_type: 'skill', skill: { id: 2, name: 'Sleight of Hand', slug: 'sleight-of-hand' } }
    ]
  }

  const wrapper = await mountSuspended(BackgroundCard, {
    props: { background }
  })

  // Should display skill names from composable
  expect(wrapper.text()).toContain('Deception, Sleight of Hand')
})
```

### Step 2: Run test to verify it passes (already working)

```bash
docker compose exec nuxt npm run test -- tests/components/background/BackgroundCard.test.ts
```

**Expected:** Test PASSES (card already shows this data, just using different logic)

### Step 3: Refactor BackgroundCard to use composable

**File:** `app/components/background/BackgroundCard.vue`

Replace lines 12-89 with composable usage:

```vue
<script setup lang="ts">
import type { Background } from '~/types'
import { ITEM_ID_GOLD_PIECE } from '~/constants/items'

interface Props {
  background: Background
}

const props = defineProps<Props>()

// Use composable for proficiency/language/equipment parsing
const backgroundRef = toRef(props, 'background')
const {
  skillProficiencies,
  toolProficiencies,
  languages,
  equipmentCount,
  startingGold
} = useBackgroundStats(backgroundRef)

/**
 * Get skill proficiencies summary - shows actual skill names
 */
const skillsSummary = computed(() => {
  if (skillProficiencies.value.length === 0) return null

  // Show first 2 skill names
  if (skillProficiencies.value.length <= 2) {
    return skillProficiencies.value.join(', ')
  }

  // Show first 2 + overflow count
  const remaining = skillProficiencies.value.length - 2
  return `${skillProficiencies.value.slice(0, 2).join(', ')} +${remaining} more`
})

/**
 * Get languages count
 */
const languagesCount = computed(() => {
  return languages.value.length > 0 ? languages.value.length : null
})

/**
 * Truncate description to specified length
 */
const truncatedDescription = computed(() => {
  if (!props.background.description) return 'A character background for D&D 5e'
  const maxLength = 150
  if (props.background.description.length <= maxLength) return props.background.description
  return props.background.description.substring(0, maxLength).trim() + '...'
})

/**
 * Get background image path (256px variant)
 */
const { getImagePath } = useEntityImage()
const backgroundImage = computed(() => {
  return getImagePath('backgrounds', props.background.slug, 256)
})
</script>
```

Update template to use `toolProficiencies.value.length` instead of manual filtering:

```vue
<!-- Tool Proficiencies -->
<div
  v-if="toolProficiencies.length > 0"
  class="flex items-center gap-2"
>
  <UBadge
    color="background"
    variant="subtle"
    size="md"
  >
    🔧 {{ toolProficiencies.length }} Tools
  </UBadge>
</div>

<!-- Equipment and Gold -->
<div
  v-if="equipmentCount || startingGold"
  class="flex items-center gap-2 flex-wrap"
>
  <UBadge
    v-if="equipmentCount"
    color="background"
    variant="subtle"
    size="md"
  >
    🎒 {{ equipmentCount }} Items
  </UBadge>
  <UBadge
    v-if="startingGold"
    color="background"
    variant="subtle"
    size="md"
  >
    💰 {{ startingGold }} gp
  </UBadge>
</div>
```

### Step 4: Run tests to verify BackgroundCard still works

```bash
docker compose exec nuxt npm run test -- tests/components/background/BackgroundCard.test.ts
```

**Expected:** All existing tests PASS (behavior unchanged)

### Step 5: Visual verification

```bash
docker compose up -d
```

Visit: `http://localhost:3000/backgrounds`

**Expected:** All background cards display correctly (skills, languages, tools, equipment, gold)

### Step 6: Commit

```bash
git add app/components/background/BackgroundCard.vue
git commit -m "refactor: Migrate BackgroundCard to use useBackgroundStats composable

- Removes duplicate proficiency/equipment parsing logic
- Uses shared composable for consistency
- Behavior unchanged, all tests passing
- 76 lines of code removed

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Add Quick Stats Panel to Detail Page (TDD)

**Files:**
- Modify: `app/pages/backgrounds/[slug].vue:61-70`
- Modify: `tests/pages/backgrounds/slug.test.ts` (add new tests)

### Step 1: Write failing tests for quick stats panel

**File:** `tests/pages/backgrounds/slug.test.ts`

Add these tests at the end of the file:

```typescript
describe('Quick Stats Panel', () => {
  it('renders quick stats panel with grid layout', async () => {
    const background = {
      id: 1,
      slug: 'charlatan',
      name: 'Charlatan',
      proficiencies: [
        { proficiency_type: 'skill', skill: { id: 1, name: 'Deception', slug: 'deception' } }
      ],
      languages: [
        { id: 1, name: 'Common', slug: 'common' }
      ]
    }

    mockApiFetch(background)

    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    // Grid layout should exist
    const grid = wrapper.find('.grid.grid-cols-1.lg\\:grid-cols-3')
    expect(grid.exists()).toBe(true)
  })

  it('shows skill proficiencies in stats panel', async () => {
    const background = {
      id: 1,
      slug: 'charlatan',
      name: 'Charlatan',
      proficiencies: [
        { proficiency_type: 'skill', skill: { id: 1, name: 'Deception', slug: 'deception' } },
        { proficiency_type: 'skill', skill: { id: 2, name: 'Sleight of Hand', slug: 'sleight-of-hand' } }
      ]
    }

    mockApiFetch(background)

    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    expect(wrapper.text()).toContain('Skill Proficiencies')
    expect(wrapper.text()).toContain('Deception, Sleight of Hand')
  })

  it('shows languages in stats panel', async () => {
    const background = {
      id: 1,
      slug: 'acolyte',
      name: 'Acolyte',
      languages: [
        { id: 1, name: 'Common', slug: 'common' },
        { id: 2, name: 'Celestial', slug: 'celestial' }
      ]
    }

    mockApiFetch(background)

    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/acolyte'
    })

    expect(wrapper.text()).toContain('Languages')
    expect(wrapper.text()).toContain('Common, Celestial')
  })

  it('shows tool proficiencies when present', async () => {
    const background = {
      id: 1,
      slug: 'charlatan',
      name: 'Charlatan',
      proficiencies: [
        { proficiency_type: 'tool', tool: { id: 1, name: 'Disguise Kit' } },
        { proficiency_type: 'tool', tool: { id: 2, name: 'Forgery Kit' } }
      ]
    }

    mockApiFetch(background)

    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    expect(wrapper.text()).toContain('Tool Proficiencies')
    expect(wrapper.text()).toContain('Disguise Kit, Forgery Kit')
  })

  it('shows equipment and gold combined', async () => {
    const background = {
      id: 1,
      slug: 'acolyte',
      name: 'Acolyte',
      equipment: [
        { item_id: 100, item_name: 'Holy Symbol', quantity: 1 },
        { item_id: 101, item_name: 'Prayer Book', quantity: 1 },
        { item_id: 102, item_name: 'Incense', quantity: 5 },
        { item_id: 1, item_name: 'Gold Piece', quantity: 15 }
      ]
    }

    mockApiFetch(background)

    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/acolyte'
    })

    expect(wrapper.text()).toContain('Starting Equipment')
    expect(wrapper.text()).toContain('3 items + 15 gp')
  })

  it('hides stats sections when data missing', async () => {
    const background = {
      id: 1,
      slug: 'test',
      name: 'Test Background',
      proficiencies: [
        { proficiency_type: 'skill', skill: { id: 1, name: 'Insight', slug: 'insight' } }
      ]
      // No languages, no tools, no equipment
    }

    mockApiFetch(background)

    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/test'
    })

    expect(wrapper.text()).toContain('Skill Proficiencies')
    expect(wrapper.text()).not.toContain('Languages')
    expect(wrapper.text()).not.toContain('Tool Proficiencies')
    expect(wrapper.text()).not.toContain('Starting Equipment')
  })

  it('renders image at 1/3 width when present', async () => {
    const background = {
      id: 1,
      slug: 'charlatan',
      name: 'Charlatan',
      proficiencies: []
    }

    mockApiFetch(background)

    const wrapper = await mountSuspended(BackgroundsSlugPage, {
      route: '/backgrounds/charlatan'
    })

    // Image column should have lg:col-span-1
    const imageColumn = wrapper.find('.lg\\:col-span-1')
    expect(imageColumn.exists()).toBe(true)
  })
})
```

### Step 2: Run tests to verify they fail

```bash
docker compose exec nuxt npm run test -- tests/pages/backgrounds/slug.test.ts
```

**Expected:** New tests FAIL (quick stats panel not yet implemented)

### Step 3: Implement quick stats panel in detail page

**File:** `app/pages/backgrounds/[slug].vue`

Add composable and stats computation after line 28:

```vue
<script setup lang="ts">
import type { Background } from '~/types/api/entities'

const route = useRoute()

// Fetch background data and setup SEO
const { data: entity, loading, error } = useEntityDetail<Background>({
  slug: route.params.slug as string,
  endpoint: '/backgrounds',
  cacheKey: 'background',
  seo: {
    titleTemplate: name => `${name} - D&D 5e Background`,
    descriptionExtractor: (background: unknown) => {
      const b = background as { description?: string }
      return b.description?.substring(0, 160) || ''
    },
    fallbackTitle: 'Background - D&D 5e Compendium'
  }
})

/**
 * Get entity image path (512px variant)
 */
const { getImagePath } = useEntityImage()
const imagePath = computed(() => {
  if (!entity.value) return null
  return getImagePath('backgrounds', entity.value.slug, 512)
})

/**
 * Use composable to extract background stats
 */
const {
  skillProficiencies,
  toolProficiencies,
  languages,
  equipmentCount,
  startingGold
} = useBackgroundStats(entity)

/**
 * Build stats array for UiDetailQuickStatsCard
 */
const statsForDisplay = computed(() => {
  const stats = []

  // Skills (always show if present)
  if (skillProficiencies.value.length > 0) {
    stats.push({
      icon: 'i-heroicons-academic-cap',
      label: 'Skill Proficiencies',
      value: skillProficiencies.value.join(', ')
    })
  }

  // Languages (show actual names)
  if (languages.value.length > 0) {
    stats.push({
      icon: 'i-heroicons-language',
      label: 'Languages',
      value: languages.value.join(', ')
    })
  }

  // Tool Proficiencies (show actual names)
  if (toolProficiencies.value.length > 0) {
    stats.push({
      icon: 'i-heroicons-wrench',
      label: 'Tool Proficiencies',
      value: toolProficiencies.value.join(', ')
    })
  }

  // Equipment + Gold combined
  if (equipmentCount.value > 0 || startingGold.value) {
    const parts = []
    if (equipmentCount.value > 0) parts.push(`${equipmentCount.value} items`)
    if (startingGold.value) parts.push(`${startingGold.value} gp`)

    stats.push({
      icon: 'i-heroicons-shopping-bag',
      label: 'Starting Equipment',
      value: parts.join(' + ')
    })
  }

  return stats
})
</script>
```

Replace the full-width image section (lines 61-70) with grid layout:

```vue
<!-- Quick Stats (2/3) + Image (1/3) Side-by-Side -->
<div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
  <!-- Quick Stats - 2/3 width on large screens -->
  <div class="lg:col-span-2">
    <UiDetailQuickStatsCard
      :columns="2"
      :stats="statsForDisplay"
    />
  </div>

  <!-- Standalone Image - 1/3 width on large screens -->
  <div class="lg:col-span-1">
    <UiDetailStandaloneImage
      v-if="imagePath"
      :image-path="imagePath"
      :image-alt="`${entity.name} background illustration`"
    />
  </div>
</div>
```

### Step 4: Run tests to verify they pass

```bash
docker compose exec nuxt npm run test -- tests/pages/backgrounds/slug.test.ts
```

**Expected:** All new tests PASS

### Step 5: Run full test suite

```bash
docker compose exec nuxt npm run test
```

**Expected:** All tests PASS (baseline + composable + card + page)

### Step 6: Visual verification

Visit: `http://localhost:3000/backgrounds/charlatan`

**Expected:**
- ✅ Grid layout: Stats on left (2/3), image on right (1/3)
- ✅ Skills displayed: "Deception, Sleight of Hand"
- ✅ Languages displayed: Actual names (if present)
- ✅ Tools displayed: "Disguise Kit, Forgery Kit"
- ✅ Equipment: "5 items + 15 gp" format
- ✅ Mobile: Stacks vertically (stats above image)

Test multiple backgrounds:
- `http://localhost:3000/backgrounds/acolyte` (different skills/languages)
- `http://localhost:3000/backgrounds/sage` (different proficiencies)

### Step 7: Commit

```bash
git add app/pages/backgrounds/[slug].vue tests/pages/backgrounds/slug.test.ts
git commit -m "feat: Add quick stats panel to backgrounds detail page

- 2/3 quick stats + 1/3 image grid layout
- Shows actual skill/tool/language names (not counts)
- Combines equipment count + gold in single stat
- Matches classes/races detail page pattern
- 7 new tests, all passing
- Mobile responsive (stacks vertically)

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Update CHANGELOG

**Files:**
- Modify: `CHANGELOG.md:1-50`

### Step 1: Add entries to CHANGELOG

**File:** `CHANGELOG.md`

Add to the top of the file:

```markdown
### Added
- Quick stats panel to backgrounds detail page (2/3 layout) - shows actual proficiency/language/equipment data (2025-11-24)

### Changed
- Background detail page image resized from full-width to 1/3 width for consistency with classes/races pages (2025-11-24)
- BackgroundCard now uses `useBackgroundStats` composable (reduces code duplication) (2025-11-24)
```

### Step 2: Commit

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG for backgrounds quick stats feature

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Final Verification

**No files modified - verification only**

### Step 1: Run complete test suite

```bash
docker compose exec nuxt npm run test
```

**Expected:** All tests PASS

### Step 2: Run TypeScript type checking

```bash
docker compose exec nuxt npm run typecheck
```

**Expected:** No type errors

### Step 3: Run linter

```bash
docker compose exec nuxt npm run lint
```

**Expected:** No lint errors

### Step 4: Visual regression testing

Test all backgrounds detail pages:

```bash
for slug in charlatan acolyte sage criminal folk-hero; do
  echo "Testing: http://localhost:3000/backgrounds/$slug"
  curl -s "http://localhost:3000/backgrounds/$slug" -o /dev/null -w "HTTP %{http_code}\n"
done
```

**Expected:** All return HTTP 200

Visit in browser:
- `http://localhost:3000/backgrounds/charlatan` - Has tools + gold
- `http://localhost:3000/backgrounds/acolyte` - Has languages
- `http://localhost:3000/backgrounds/sage` - Different proficiencies

**Verify:**
- ✅ Grid layout (stats 2/3, image 1/3)
- ✅ Stats show actual names (not counts)
- ✅ Mobile responsive (stacks vertically)
- ✅ Dark mode works
- ✅ No horizontal scrollbars
- ✅ Consistent with classes/races pages

### Step 5: Check git status

```bash
git status
```

**Expected:** Clean working tree (all changes committed)

### Step 6: Review commit history

```bash
git log --oneline -5
```

**Expected:** 4 commits:
1. Update CHANGELOG
2. Add quick stats panel to detail page
3. Migrate BackgroundCard to composable
4. Create useBackgroundStats composable

---

## Success Criteria Checklist

**Functionality:**
- ✅ Quick stats panel displays at 2/3 width
- ✅ Image displays at 1/3 width
- ✅ Actual skill names shown (not counts)
- ✅ Actual language names shown (not counts)
- ✅ Actual tool names shown (not counts)
- ✅ Equipment + gold combined properly
- ✅ Missing data handled gracefully

**Code Quality:**
- ✅ Composable created with full test coverage
- ✅ BackgroundCard refactored to use composable
- ✅ No code duplication
- ✅ TypeScript types correct
- ✅ All tests passing

**Visual Consistency:**
- ✅ Matches classes/races detail page pattern
- ✅ Grid layout responsive (mobile stacks)
- ✅ Dark mode works
- ✅ No visual regressions

**Documentation:**
- ✅ CHANGELOG updated
- ✅ Composable has JSDoc comments
- ✅ All commits follow conventional commit format

---

## Rollback Plan

If issues discovered:

```bash
# Revert all commits
git reset --hard HEAD~4

# Or revert individual commits
git revert <commit-hash>
```

---

## Notes

**TDD Workflow:**
- Every task follows RED-GREEN-REFACTOR
- Write test FIRST, verify it fails
- Implement minimal code to pass
- Run tests, commit when green

**Code Reuse:**
- `useBackgroundStats` composable eliminates 76 lines of duplicate logic
- Both BackgroundCard and detail page use same composable
- Future features can reuse this composable

**Visual Consistency:**
- Exact same grid pattern as classes/races pages
- Same `UiDetailQuickStatsCard` and `UiDetailStandaloneImage` components
- Maintains project's design system

**Performance:**
- No new API calls (uses existing entity data)
- Composable uses computed properties (reactive, cached)
- Grid layout uses Tailwind (no runtime CSS)

---

**Plan Status:** Ready for execution
**Estimated Time:** 60-90 minutes (with TDD, testing, verification)
**Next Step:** Use `superpowers:executing-plans` or `superpowers:subagent-driven-development`
