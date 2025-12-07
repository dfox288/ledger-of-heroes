# Search Page Entity Cards Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Replace generic SearchResultCard with specialized entity-specific cards and add monster support to search.

**Architecture:** Direct component replacement approach—import all 7 specialized card components (SpellCard, ItemCard, RaceCard, ClassCard, BackgroundCard, FeatCard, MonsterCard) and use them directly in search.vue template. Add Monster type to search type system.

**Tech Stack:** Nuxt 4, Vue 3 Composition API, TypeScript, Vitest, @nuxt/test-utils

---

## Task 1: Add Monster Type to Search Types

**Files:**
- Modify: `app/types/search.ts`
- Test: `tests/types/search.test.ts` (create if doesn't exist)

**Step 1: Write the failing test**

Create `tests/types/search.test.ts`:

```typescript
import { describe, it, expect } from 'vitest'
import type { SearchResultData, EntityType } from '~/types/search'

describe('Search Types', () => {
  it('supports monsters in SearchResultData', () => {
    const mockData: SearchResultData = {
      spells: [],
      items: [],
      races: [],
      classes: [],
      backgrounds: [],
      feats: [],
      monsters: [] // Should not cause TypeScript error
    }

    expect(mockData.monsters).toBeDefined()
  })

  it('includes monsters in EntityType', () => {
    const entityTypes: EntityType[] = [
      'spells',
      'items',
      'races',
      'classes',
      'backgrounds',
      'feats',
      'monsters' // Should not cause TypeScript error
    ]

    expect(entityTypes).toContain('monsters')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/types/search.test.ts`

Expected: TypeScript error - "Type '{ monsters: never[] }' is not assignable to type 'SearchResultData'"

**Step 3: Write minimal implementation**

In `app/types/search.ts`, add Monster import and type:

```typescript
import type {
  Spell as SpellEntity,
  Item as ItemEntity,
  Race as RaceEntity,
  CharacterClass as CharacterClassEntity,
  Background as BackgroundEntity,
  Feat as FeatEntity,
  Monster as MonsterEntity  // ADD THIS
} from './api/entities'

// Re-export entity types
export type Spell = SpellEntity
export type Item = ItemEntity
export type Race = RaceEntity
export type CharacterClass = CharacterClassEntity
export type Background = BackgroundEntity
export type Feat = FeatEntity
export type Monster = MonsterEntity  // ADD THIS
```

Update EntityType:

```typescript
export type EntityType = 'spells' | 'items' | 'races' | 'classes' | 'backgrounds' | 'feats' | 'monsters'  // Add 'monsters'
```

Update SearchResultData:

```typescript
export interface SearchResultData {
  spells?: Spell[]
  items?: Item[]
  races?: Race[]
  classes?: CharacterClass[]
  backgrounds?: Background[]
  feats?: Feat[]
  monsters?: Monster[]  // ADD THIS
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/types/search.test.ts`

Expected: PASS (2 tests)

**Step 5: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No errors

**Step 6: Commit**

```bash
git add app/types/search.ts tests/types/search.test.ts
git commit -m "feat: Add Monster type to search types

- Import Monster type from entities
- Add 'monsters' to EntityType union
- Add monsters array to SearchResultData
- Add type tests for monster support

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 2: Test Monster Filter Button in Search Page

**Files:**
- Test: `tests/components/pages/search.test.ts` (create if doesn't exist)

**Step 1: Write the failing test**

Create `tests/components/pages/search.test.ts`:

```typescript
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SearchPage from '~/pages/search.vue'

// Mock the useSearch composable
vi.mock('~/composables/useSearch', () => ({
  useSearch: () => ({
    results: ref({
      data: {
        spells: [{ id: 1, name: 'Fireball', slug: 'fireball', description: 'Test', level: 3 }],
        monsters: [
          { id: 1, name: 'Dragon', slug: 'dragon', description: 'Test', type: 'Dragon', challenge_rating: '5', armor_class: 17, hit_points_average: 100 },
          { id: 2, name: 'Goblin', slug: 'goblin', description: 'Test', type: 'Humanoid', challenge_rating: '1/4', armor_class: 15, hit_points_average: 7 }
        ]
      }
    }),
    loading: ref(false),
    error: ref(null),
    search: vi.fn()
  })
}))

describe('Search Page - Monster Support', () => {
  it('includes Monsters filter button', async () => {
    const wrapper = await mountSuspended(SearchPage, {
      route: { query: { q: 'test' } }
    })

    const filterButtons = wrapper.findAll('button')
    const monsterButton = filterButtons.find(btn => btn.text().includes('Monsters'))

    expect(monsterButton).toBeDefined()
    expect(monsterButton?.text()).toContain('Monsters (2)')
  })

  it('displays correct count for monsters', async () => {
    const wrapper = await mountSuspended(SearchPage, {
      route: { query: { q: 'test' } }
    })

    const filterButtons = wrapper.findAll('button')
    const monsterButton = filterButtons.find(btn => btn.text().includes('Monsters'))

    expect(monsterButton?.text()).toBe('Monsters (2)')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/pages/search.test.ts`

Expected: FAIL - Monster button not found

**Step 3: Update search page to include monster filter**

In `app/pages/search.vue`, update `filterOptions`:

```typescript
const filterOptions = computed(() => [
  { label: `All (${totalCount.value})`, value: 'all' },
  { label: `Spells (${getCount('spells')})`, value: 'spells', disabled: getCount('spells') === 0 },
  { label: `Items (${getCount('items')})`, value: 'items', disabled: getCount('items') === 0 },
  { label: `Races (${getCount('races')})`, value: 'races', disabled: getCount('races') === 0 },
  { label: `Classes (${getCount('classes')})`, value: 'classes', disabled: getCount('classes') === 0 },
  { label: `Backgrounds (${getCount('backgrounds')})`, value: 'backgrounds', disabled: getCount('backgrounds') === 0 },
  { label: `Feats (${getCount('feats')})`, value: 'feats', disabled: getCount('feats') === 0 },
  { label: `Monsters (${getCount('monsters')})`, value: 'monsters', disabled: getCount('monsters') === 0 }  // ADD THIS LINE
])
```

Update `getFilterColor`:

```typescript
const getFilterColor = (value: string): BadgeColor => {
  const entityColors: Record<string, BadgeColor> = {
    spells: 'spell',
    items: 'item',
    races: 'race',
    classes: 'class',
    backgrounds: 'background',
    feats: 'feat',
    monsters: 'monster'  // ADD THIS LINE
  }
  return entityColors[value] || 'neutral'
}
```

**Step 4: Run test to verify it passes**

Run: `docker compose exec nuxt npm run test -- tests/components/pages/search.test.ts`

Expected: PASS (2 tests)

**Step 5: Commit**

```bash
git add app/pages/search.vue tests/components/pages/search.test.ts
git commit -m "feat: Add monster filter button to search page

- Add Monsters to filter options with count
- Add monster semantic color to filter mapping
- Add tests for monster filter button

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 3: Test Specialized Card Rendering

**Files:**
- Test: `tests/components/pages/search.test.ts`

**Step 1: Write the failing tests**

Add to `tests/components/pages/search.test.ts`:

```typescript
import SpellCard from '~/components/spell/SpellCard.vue'
import ItemCard from '~/components/item/ItemCard.vue'
import MonsterCard from '~/components/monster/MonsterCard.vue'

describe('Search Page - Specialized Card Rendering', () => {
  it('renders SpellCard for spell results', async () => {
    const wrapper = await mountSuspended(SearchPage, {
      route: { query: { q: 'fireball' } }
    })

    const spellCard = wrapper.findComponent(SpellCard)
    expect(spellCard.exists()).toBe(true)
    expect(spellCard.props('spell')).toMatchObject({
      id: 1,
      name: 'Fireball'
    })
  })

  it('renders MonsterCard for monster results', async () => {
    const wrapper = await mountSuspended(SearchPage, {
      route: { query: { q: 'dragon' } }
    })

    const monsterCards = wrapper.findAllComponents(MonsterCard)
    expect(monsterCards).toHaveLength(2)
    expect(monsterCards[0].props('monster')).toMatchObject({
      id: 1,
      name: 'Dragon'
    })
  })

  it('does not render SearchResultCard component', async () => {
    const wrapper = await mountSuspended(SearchPage, {
      route: { query: { q: 'test' } }
    })

    // SearchResultCard should not be used anymore
    const html = wrapper.html()
    expect(html).not.toContain('SearchResultCard')
  })
})
```

**Step 2: Run test to verify it fails**

Run: `docker compose exec nuxt npm run test -- tests/components/pages/search.test.ts`

Expected: FAIL - SpellCard/MonsterCard not found

**Step 3: Import specialized cards in search page**

In `app/pages/search.vue`, update imports section:

```typescript
<script setup lang="ts">
import { ref, watch, computed } from 'vue'
import type { EntityType, SearchResultData } from '~/types/search'
import type { BadgeColor } from '~/utils/badgeColors'

// Import all specialized entity cards
import SpellCard from '~/components/spell/SpellCard.vue'
import ItemCard from '~/components/item/ItemCard.vue'
import RaceCard from '~/components/race/RaceCard.vue'
import ClassCard from '~/components/class/ClassCard.vue'
import BackgroundCard from '~/components/background/BackgroundCard.vue'
import FeatCard from '~/components/feat/FeatCard.vue'
import MonsterCard from '~/components/monster/MonsterCard.vue'

// ... rest of script
```

**Step 4: Replace SearchResultCard with SpellCard in template**

In `app/pages/search.vue` template, replace the Spells section:

```vue
<!-- Spells -->
<div v-if="filteredResults.spells && filteredResults.spells.length > 0">
  <h2 class="text-2xl font-bold text-gray-900 dark:text-gray-100 mb-4">
    Spells ({{ filteredResults.spells.length }})
  </h2>
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    <SpellCard
      v-for="spell in filteredResults.spells"
      :key="spell.id"
      :spell="spell"
    />
  </div>
</div>
```

**Step 5: Replace SearchResultCard with ItemCard**

Replace the Items section:

```vue
<!-- Items -->
<div v-if="filteredResults.items && filteredResults.items.length > 0">
  <h2 class="text-2xl font-bold text-gray-900 dark:text-gray-100 mb-4">
    Items ({{ filteredResults.items.length }})
  </h2>
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    <ItemCard
      v-for="item in filteredResults.items"
      :key="item.id"
      :item="item"
    />
  </div>
</div>
```

**Step 6: Replace SearchResultCard with RaceCard**

Replace the Races section:

```vue
<!-- Races -->
<div v-if="filteredResults.races && filteredResults.races.length > 0">
  <h2 class="text-2xl font-bold text-gray-900 dark:text-gray-100 mb-4">
    Races ({{ filteredResults.races.length }})
  </h2>
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    <RaceCard
      v-for="race in filteredResults.races"
      :key="race.id"
      :race="race"
    />
  </div>
</div>
```

**Step 7: Replace SearchResultCard with ClassCard**

Replace the Classes section:

```vue
<!-- Classes -->
<div v-if="filteredResults.classes && filteredResults.classes.length > 0">
  <h2 class="text-2xl font-bold text-gray-900 dark:text-gray-100 mb-4">
    Classes ({{ filteredResults.classes.length }})
  </h2>
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    <ClassCard
      v-for="charClass in filteredResults.classes"
      :key="charClass.id"
      :character-class="charClass"
    />
  </div>
</div>
```

**Step 8: Replace SearchResultCard with BackgroundCard**

Replace the Backgrounds section:

```vue
<!-- Backgrounds -->
<div v-if="filteredResults.backgrounds && filteredResults.backgrounds.length > 0">
  <h2 class="text-2xl font-bold text-gray-900 dark:text-gray-100 mb-4">
    Backgrounds ({{ filteredResults.backgrounds.length }})
  </h2>
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    <BackgroundCard
      v-for="background in filteredResults.backgrounds"
      :key="background.id"
      :background="background"
    />
  </div>
</div>
```

**Step 9: Replace SearchResultCard with FeatCard**

Replace the Feats section:

```vue
<!-- Feats -->
<div v-if="filteredResults.feats && filteredResults.feats.length > 0">
  <h2 class="text-2xl font-bold text-gray-900 dark:text-gray-100 mb-4">
    Feats ({{ filteredResults.feats.length }})
  </h2>
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    <FeatCard
      v-for="feat in filteredResults.feats"
      :key="feat.id"
      :feat="feat"
    />
  </div>
</div>
```

**Step 10: Add new Monsters section with MonsterCard**

Add after the Feats section:

```vue
<!-- Monsters -->
<div v-if="filteredResults.monsters && filteredResults.monsters.length > 0">
  <h2 class="text-2xl font-bold text-gray-900 dark:text-gray-100 mb-4">
    Monsters ({{ filteredResults.monsters.length }})
  </h2>
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
    <MonsterCard
      v-for="monster in filteredResults.monsters"
      :key="monster.id"
      :monster="monster"
    />
  </div>
</div>
```

**Step 11: Run tests to verify they pass**

Run: `docker compose exec nuxt npm run test -- tests/components/pages/search.test.ts`

Expected: PASS (all tests)

**Step 12: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No errors

**Step 13: Commit**

```bash
git add app/pages/search.vue tests/components/pages/search.test.ts
git commit -m "feat: Replace SearchResultCard with specialized entity cards

- Import all 7 specialized card components
- Replace generic SearchResultCard with entity-specific cards
- Add MonsterCard for monster search results
- Add tests for specialized card rendering

Each entity type now uses its dedicated card component with
full metadata, background images, and semantic styling.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 4: Delete SearchResultCard Component

**Files:**
- Delete: `app/components/SearchResultCard.vue`
- Test: Verify no remaining references

**Step 1: Check for remaining references**

Run: `docker compose exec nuxt sh -c "grep -r 'SearchResultCard' app/ || echo 'No references found'"`

Expected: "No references found" (we already replaced all usages)

**Step 2: Delete the component**

```bash
git rm app/components/SearchResultCard.vue
```

**Step 3: Run full test suite**

Run: `docker compose exec nuxt npm run test`

Expected: All tests PASS

**Step 4: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No errors

**Step 5: Commit**

```bash
git commit -m "refactor: Delete redundant SearchResultCard component

Component is now fully replaced by specialized entity cards.
All search results use entity-specific components (SpellCard,
ItemCard, MonsterCard, etc.) for richer metadata display.

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 5: Manual Browser Testing

**Files:**
- None (manual verification)

**Step 1: Start Docker environment**

Run: `docker compose up -d`

Expected: All services start successfully

**Step 2: Verify backend is running**

Run: `curl -s http://localhost:8080/api/v1/spells | head -20`

Expected: JSON response with spell data

**Step 3: Access frontend**

Open browser: `http://localhost:3000`

Expected: Homepage loads

**Step 4: Test search functionality**

1. Navigate to search: `http://localhost:3000/search?q=dragon`
2. Verify monster results appear with MonsterCard styling
3. Verify CR badge, type badge, AC/HP stats display
4. Verify background image displays with hover effect

**Step 5: Test entity-specific cards**

Search for different entities:
- `?q=fireball` - verify SpellCard shows level, school, concentration
- `?q=sword` - verify ItemCard shows rarity, cost, magic badges
- `?q=elf` - verify RaceCard shows size, speed, traits
- `?q=wizard` - verify ClassCard shows hit die, abilities

**Step 6: Test filter buttons**

1. Verify "Monsters" filter button appears
2. Click "Monsters" filter
3. Verify only MonsterCard components render
4. Verify monster filter uses semantic 'monster' color

**Step 7: Test dark mode**

1. Toggle dark mode
2. Verify all cards display correctly
3. Verify border colors adapt (border-monster-700 in dark)
4. Verify text contrast is readable

**Step 8: Test mobile responsiveness**

1. Resize browser to 375px width
2. Verify cards stack in single column
3. Verify all metadata remains readable
4. Test on 768px (tablet) - should show 2 columns
5. Test on 1440px (desktop) - should show 3 columns

**Step 9: Document any issues**

If issues found, create GitHub issues or fix immediately.

Expected: All manual tests PASS

---

## Task 6: Update CHANGELOG

**Files:**
- Modify: `CHANGELOG.md`

**Step 1: Add entry to CHANGELOG**

Add to top of `CHANGELOG.md` under `### Added` or create new dated section:

```markdown
### Added
- Monster support in search results (2025-11-23)
- Specialized entity cards in search (SpellCard, ItemCard, MonsterCard, etc.) (2025-11-23)

### Changed
- Search results now use entity-specific card components instead of generic SearchResultCard (2025-11-23)
- Search page visual consistency now matches list pages with semantic colors and background images (2025-11-23)

### Removed
- SearchResultCard component (replaced by specialized entity cards) (2025-11-23)
```

**Step 2: Commit**

```bash
git add CHANGELOG.md
git commit -m "docs: Update CHANGELOG for search page improvements

🤖 Generated with [Claude Code](https://claude.com/claude-code)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Task 7: Final Verification

**Files:**
- None (verification only)

**Step 1: Run full test suite**

Run: `docker compose exec nuxt npm run test`

Expected: All tests PASS

**Step 2: Run typecheck**

Run: `docker compose exec nuxt npm run typecheck`

Expected: No TypeScript errors

**Step 3: Run linter**

Run: `docker compose exec nuxt npm run lint`

Expected: No linting errors (or only warnings)

**Step 4: Build production bundle**

Run: `docker compose exec nuxt npm run build`

Expected: Build succeeds with no errors

**Step 5: Verify git status**

Run: `git status`

Expected: Working tree clean (all changes committed)

**Step 6: Review commit history**

Run: `git log --oneline -7`

Expected: 7 new commits for this feature

**Step 7: Create summary**

Success! Search page now uses specialized entity cards with monster support. All tests pass, TypeScript compiles, and manual browser testing confirms visual consistency with list pages.

---

## Success Checklist

- ✅ Monster type added to search types with tests
- ✅ Monster filter button added with correct count
- ✅ All 7 specialized cards imported and used
- ✅ SearchResultCard component deleted
- ✅ All tests pass (unit + integration)
- ✅ TypeScript compiles with no errors
- ✅ Manual browser verification complete (light/dark mode, responsive)
- ✅ CHANGELOG updated
- ✅ All changes committed with descriptive messages
- ✅ Production build succeeds

---

## Rollback Plan

If critical issues found:

```bash
# Revert all commits from this feature
git log --oneline -7  # Note the commit hash before this feature
git revert --no-commit <hash>..HEAD
git commit -m "revert: Rollback search page entity cards feature"
```

Or revert specific commits:

```bash
git revert <commit-hash>
```
