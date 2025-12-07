# Search Page Entity Cards Design

**Date:** 2025-11-23
**Status:** Approved
**Approach:** Direct Component Replacement

---

## Problem Statement

The search page (`app/pages/search.vue`) currently uses a generic `SearchResultCard` component that:
- Only displays minimal metadata for spells and items
- Shows no entity-specific information for races, classes, backgrounds, or feats
- Lacks the visual polish of specialized entity cards used on list pages
- Does not support monsters in search results
- Doesn't leverage background images, semantic colors, or rich badges

This creates a jarring UX inconsistency—users see beautiful, information-rich cards on list pages but get bare-bones cards in search results.

---

## Goals

1. **Visual Consistency:** Search results should match the quality of list page cards
2. **Entity-Specific UI:** Each entity type shows its relevant metadata (spell schools, item rarity, class hit dice, etc.)
3. **Monster Support:** Add monsters to search functionality
4. **Code Reuse:** Use existing specialized card components instead of maintaining a separate generic card
5. **Maintainability:** Reduce component count by deleting redundant code

---

## Solution: Direct Component Replacement

### Architecture Decision

**Chosen Approach:** Import and use all 7 specialized card components directly in the search page.

**Why this approach:**
- Search page already follows this pattern (just uses wrong component)
- Each card component is used exactly as designed—no abstraction needed
- TypeScript provides full type safety with explicit imports
- Easy to debug and maintain
- Matches existing codebase patterns from list pages

**Rejected Alternatives:**
- Dynamic `<component :is>` wrapper: Adds unnecessary abstraction and TypeScript complexity
- Type guard approach: Overkill for already well-typed data structures

---

## Implementation Details

### 1. Type System Updates

**File:** `app/types/search.ts`

Add `Monster` support:

```typescript
import type {
  Spell as SpellEntity,
  Item as ItemEntity,
  Race as RaceEntity,
  CharacterClass as CharacterClassEntity,
  Background as BackgroundEntity,
  Feat as FeatEntity,
  Monster as MonsterEntity  // NEW
} from './api/entities'

// Re-export entity types
export type Monster = MonsterEntity  // NEW

export type EntityType = 'spells' | 'items' | 'races' | 'classes' | 'backgrounds' | 'feats' | 'monsters'  // Added 'monsters'

export interface SearchResultData {
  spells?: Spell[]
  items?: Item[]
  races?: Race[]
  classes?: CharacterClass[]
  backgrounds?: Background[]
  feats?: Feat[]
  monsters?: Monster[]  // NEW
}
```

**Rationale:** Search API likely already supports monsters—we're just exposing it on frontend.

---

### 2. Search Page Component Updates

**File:** `app/pages/search.vue`

#### Imports

```typescript
// Remove this:
// import SearchResultCard from '~/components/SearchResultCard.vue'

// Add these:
import SpellCard from '~/components/spell/SpellCard.vue'
import ItemCard from '~/components/item/ItemCard.vue'
import RaceCard from '~/components/race/RaceCard.vue'
import ClassCard from '~/components/class/ClassCard.vue'
import BackgroundCard from '~/components/background/BackgroundCard.vue'
import FeatCard from '~/components/feat/FeatCard.vue'
import MonsterCard from '~/components/monster/MonsterCard.vue'
```

#### Filter Options Update

```typescript
const filterOptions = computed(() => [
  { label: `All (${totalCount.value})`, value: 'all' },
  { label: `Spells (${getCount('spells')})`, value: 'spells', disabled: getCount('spells') === 0 },
  { label: `Items (${getCount('items')})`, value: 'items', disabled: getCount('items') === 0 },
  { label: `Races (${getCount('races')})`, value: 'races', disabled: getCount('races') === 0 },
  { label: `Classes (${getCount('classes')})`, value: 'classes', disabled: getCount('classes') === 0 },
  { label: `Backgrounds (${getCount('backgrounds')})`, value: 'backgrounds', disabled: getCount('backgrounds') === 0 },
  { label: `Feats (${getCount('feats')})`, value: 'feats', disabled: getCount('feats') === 0 },
  { label: `Monsters (${getCount('monsters')})`, value: 'monsters', disabled: getCount('monsters') === 0 }  // NEW
])
```

#### Filter Color Mapping

```typescript
const getFilterColor = (value: string): BadgeColor => {
  const entityColors: Record<string, BadgeColor> = {
    spells: 'spell',
    items: 'item',
    races: 'race',
    classes: 'class',
    backgrounds: 'background',
    feats: 'feat',
    monsters: 'monster'  // NEW
  }
  return entityColors[value] || 'neutral'
}
```

#### Template Replacement

Replace all `<SearchResultCard>` instances with entity-specific cards:

```vue
<!-- Spells Section -->
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

<!-- Items Section -->
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

<!-- Races Section -->
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

<!-- Classes Section -->
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

<!-- Backgrounds Section -->
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

<!-- Feats Section -->
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

<!-- Monsters Section (NEW) -->
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

---

### 3. Component Cleanup

**Delete:** `app/components/SearchResultCard.vue`

This component becomes redundant. Its functionality is fully replaced by the specialized entity cards.

**Migration path:**
1. Update search page to use specialized cards
2. Verify all tests pass
3. Delete SearchResultCard.vue
4. Delete any tests specific to SearchResultCard

---

## Testing Strategy (TDD)

### Test File

**Location:** `tests/components/pages/search.test.ts` (create if doesn't exist)

### Test Coverage

#### 1. Component Rendering Tests

```typescript
describe('Search Page - Entity Card Rendering', () => {
  it('renders SpellCard for spell results', async () => {
    // Mock search results with spells
    // Verify SpellCard component is rendered
    // Verify correct props passed
  })

  it('renders ItemCard for item results', async () => {
    // Similar pattern for items
  })

  it('renders MonsterCard for monster results', async () => {
    // NEW: Test monster card rendering
  })

  // Repeat for all 7 entity types
})
```

#### 2. Filter Functionality Tests

```typescript
describe('Search Page - Filter Functionality', () => {
  it('includes monsters in filter options', async () => {
    // Verify monsters appear in filter button list
  })

  it('displays correct monster count in filter button', async () => {
    // Mock results with 5 monsters
    // Verify button shows "Monsters (5)"
  })

  it('uses monster color for monster filter button', async () => {
    // Verify button has correct semantic color
  })

  it('filters results to show only monsters when monster filter selected', async () => {
    // Select monster filter
    // Verify only MonsterCard components render
  })
})
```

#### 3. Data Flow Tests

```typescript
describe('Search Page - Data Flow', () => {
  it('passes correct props to each card type', async () => {
    // Mock search results with all 7 entity types
    // Verify SpellCard receives spell prop
    // Verify ItemCard receives item prop
    // etc.
  })

  it('handles empty results for each entity type', async () => {
    // Mock results with some empty arrays
    // Verify no cards render for empty types
  })
})
```

#### 4. Regression Tests

```typescript
describe('Search Page - Regression', () => {
  it('maintains existing search functionality', async () => {
    // Verify search query processing still works
    // Verify loading/error states still work
    // Verify "All" filter still works
  })
})
```

---

## Implementation Workflow (TDD)

### Phase 1: RED (Write Failing Tests)
1. Write tests for monster support in types
2. Write tests for MonsterCard rendering
3. Write tests for monster filter functionality
4. Write tests for specialized card prop passing
5. **Expected:** All tests fail (features not implemented)

### Phase 2: GREEN (Minimal Implementation)
1. Update `app/types/search.ts` to add Monster
2. Update `app/pages/search.vue`:
   - Import all 7 specialized cards
   - Update filter options
   - Update filter color mapping
   - Replace SearchResultCard with specialized cards
   - Add monsters section to template
3. **Expected:** All tests pass

### Phase 3: REFACTOR
1. Delete `app/components/SearchResultCard.vue`
2. Delete SearchResultCard tests (if any)
3. Run full test suite
4. **Expected:** All tests still pass

### Phase 4: VERIFY
1. Start Docker environment: `docker compose up -d`
2. Run dev server: `docker compose exec nuxt npm run dev`
3. Manual browser testing:
   - Search for "dragon" → verify monsters appear
   - Test each entity type filter
   - Verify cards match list page styling
   - Test light/dark mode
   - Test mobile responsiveness
4. Run full test suite: `docker compose exec nuxt npm run test`
5. Run type checking: `docker compose exec nuxt npm run typecheck`
6. **Expected:** All checks pass

### Phase 5: COMMIT
1. Commit with passing tests
2. Update CHANGELOG.md

---

## Success Criteria

- ✅ Search results display entity-specific cards with full metadata
- ✅ All 7 entity types (including monsters) render correctly
- ✅ Background images display with hover effects
- ✅ Semantic colors match list pages (border-spell-300, etc.)
- ✅ Filter buttons include monsters with correct count
- ✅ Monster filter color uses semantic 'monster' color
- ✅ All tests pass (unit + integration)
- ✅ TypeScript compiles with no errors
- ✅ Manual browser verification in Docker (light/dark mode)
- ✅ SearchResultCard.vue deleted
- ✅ Code committed with tests passing

---

## Rollback Plan

If issues arise:

1. **Revert commit:** `git revert <commit-hash>`
2. **Restore SearchResultCard.vue** from git history
3. **Remove monster types** from search.ts
4. **Investigate issue** before re-attempting

---

## Future Enhancements

(Out of scope for this design)

- Add sorting options (by name, relevance, etc.)
- Add pagination for large result sets
- Add "view all" links for each entity type
- Add keyboard navigation between results
- Add analytics tracking for search queries
