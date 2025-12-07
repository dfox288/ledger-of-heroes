# Test Suite Optimization Plan

**Created:** 2025-11-29
**Completed:** 2025-11-29
**Status:** ✅ Complete
**Goal:** Reduce test suite runtime from ~125s to ~75-85s (30-40% improvement)

## Results Summary

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| `nextTick()` calls | 316 | 205 | -111 (35%) |
| `for` loops in tests | 11 | 0 | -100% converted to `it.each()` |
| Redundant card mounts | ~100+ | ~30 | -70+ mounts eliminated |
| Test files | 133 | 137 | +4 (better organization) |
| Monster filter file | 774 lines | 4 focused files | Split for maintainability |

**Commits:**
- `ce8c86b` - Wave 1: it.each conversions + initial nextTick reductions
- `92624cd` - Wave 2: nextTick reductions across 10 more files
- `ad2c69e` - Wave 3: SharedCardBehavior.test.ts + monster filter split + 17 card cleanups

**Agents dispatched:** 42 parallel subagents across 3 waves

---

## Executive Summary

The test suite has grown to 147 test files with a 125-second runtime. Analysis reveals 25-50 seconds of waste from:
- Redundant component mounts (no deduplication)
- Missing parameterization (no `it.each()` usage)
- Inefficient async patterns (excessive `nextTick()`)
- Duplicated boilerplate (card tests, store tests)

This plan breaks the optimization into parallelizable work streams.

---

## Current State Metrics

| Metric | Value |
|--------|-------|
| Total test files | 147 |
| Full suite runtime | ~125 seconds |
| Card component tests | 17 files |
| Filter store tests | 7 files |
| Page filter tests | 5 files |
| Reference entity card tests | 10 files |

---

## Work Streams

### Stream 1: Convert Loops to `it.each()` [LOW EFFORT, HIGH IMPACT]

**Estimated time savings:** 5-10 seconds
**Effort:** 1-2 hours
**Parallelizable:** YES (each file independent)

Convert `for` loops inside tests to Vitest's `it.each()` for better reporting and faster failure detection.

#### Files to Update:

| File | Location | Description |
|------|----------|-------------|
| `tests/components/item/ItemCard.test.ts` | Lines 235-259 | Rarity levels, weapon types |
| `tests/components/race/RaceCard.test.ts` | ~Lines 300-400 | Size iterations |
| `tests/components/monster/MonsterCard.test.ts` | Various | CR test iterations |
| `tests/components/class/ClassCard.test.ts` | Various | Hit die, ability iterations |
| `tests/components/damage-type/DamageTypeCard.test.ts` | Various | Damage type iterations |
| `tests/components/language/LanguageCard.test.ts` | Various | Script type iterations |
| `tests/components/size/SizeCard.test.ts` | Various | Size iterations |
| `tests/components/skill/SkillCard.test.ts` | Various | Skill iterations |

#### Pattern to Apply:

**Before:**
```typescript
it('handles different rarity levels', async () => {
  const rarities = ['common', 'uncommon', 'rare', 'very rare', 'legendary', 'artifact']
  for (const rarity of rarities) {
    const wrapper = await mountSuspended(ItemCard, { props: { item: { ...mockItem, rarity } } })
    expect(wrapper.html()).toBeTruthy()
  }
})
```

**After:**
```typescript
it.each([
  ['common'],
  ['uncommon'],
  ['rare'],
  ['very rare'],
  ['legendary'],
  ['artifact'],
])('renders item with %s rarity', async (rarity) => {
  const wrapper = await mountSuspended(ItemCard, {
    props: { item: { ...mockItem, rarity } }
  })
  expect(wrapper.text()).toContain(rarity.charAt(0).toUpperCase() + rarity.slice(1))
})
```

#### Acceptance Criteria:
- [ ] All `for` loops inside `it()` blocks converted to `it.each()`
- [ ] Test output shows individual test names (e.g., "renders item with rare rarity")
- [ ] All tests still pass
- [ ] No new test failures

---

### Stream 2: Reduce Card Test Redundant Mounts [MEDIUM EFFORT, HIGH IMPACT]

**Estimated time savings:** 8-15 seconds
**Effort:** 3-4 hours
**Parallelizable:** YES (after shared helper created)

#### Phase 2A: Create Shared Card Behavior Test Suite

Create a single parameterized test file that tests shared behavior across all 17 card components.

**New file:** `tests/components/cards/SharedCardBehavior.test.ts`

```typescript
import { mountSuspended } from '@nuxt/test-utils/runtime'
import { describe, it, expect } from 'vitest'

// Import all card components
import SpellCard from '~/components/spell/SpellCard.vue'
import ItemCard from '~/components/item/ItemCard.vue'
// ... 15 more

// Import mock factories
import { createMockSpell, createMockItem, /* ... */ } from '../../helpers/mockFactories'

const cardConfigs = [
  {
    name: 'SpellCard',
    component: SpellCard,
    props: { spell: createMockSpell() },
    expectedLink: '/spells/fireball',
    entityName: 'Fireball'
  },
  {
    name: 'ItemCard',
    component: ItemCard,
    props: { item: createMockItem() },
    expectedLink: '/items/longsword',
    entityName: 'Longsword'
  },
  // ... 15 more
]

describe('Shared Card Behavior', () => {
  describe.each(cardConfigs)('$name', ({ component, props, expectedLink, entityName }) => {
    it('renders as a clickable link', async () => {
      const wrapper = await mountSuspended(component, { props })
      const link = wrapper.find('a')
      expect(link.exists()).toBe(true)
      expect(link.attributes('href')).toBe(expectedLink)
    })

    it('displays entity name', async () => {
      const wrapper = await mountSuspended(component, { props })
      expect(wrapper.text()).toContain(entityName)
    })

    it('has hover effects', async () => {
      const wrapper = await mountSuspended(component, { props })
      expect(wrapper.classes()).toContain('hover:shadow-lg')
    })

    it('has proper border styling', async () => {
      const wrapper = await mountSuspended(component, { props })
      expect(wrapper.classes().some(c => c.includes('border'))).toBe(true)
    })
  })
})
```

#### Phase 2B: Remove Shared Tests from Individual Card Files

After Phase 2A is complete, remove the shared helper calls from each card test file:

```typescript
// REMOVE these lines from each card test file:
testCardLinkBehavior(() => mountSuspended(SpellCard, { props: { spell: mockSpell } }), '/spells/fireball')
testCardHoverEffects(() => mountSuspended(SpellCard, { props: { spell: mockSpell } }))
testCardBorderStyling(() => mountSuspended(SpellCard, { props: { spell: mockSpell } }))
testCardAccessibility(() => mountSuspended(SpellCard, { props: { spell: mockSpell } }))
```

**Files to update (17 total):**
- `tests/components/spell/SpellCard.test.ts`
- `tests/components/item/ItemCard.test.ts`
- `tests/components/monster/MonsterCard.test.ts`
- `tests/components/class/ClassCard.test.ts`
- `tests/components/race/RaceCard.test.ts`
- `tests/components/background/BackgroundCard.test.ts`
- `tests/components/feat/FeatCard.test.ts`
- `tests/components/ability-score/AbilityScoreCard.test.ts`
- `tests/components/condition/ConditionCard.test.ts`
- `tests/components/damage-type/DamageTypeCard.test.ts`
- `tests/components/item-type/ItemTypeCard.test.ts`
- `tests/components/language/LanguageCard.test.ts`
- `tests/components/proficiency-type/ProficiencyTypeCard.test.ts`
- `tests/components/size/SizeCard.test.ts`
- `tests/components/skill/SkillCard.test.ts`
- `tests/components/source/SourceCard.test.ts`
- `tests/components/spell-school/SpellSchoolCard.test.ts`

#### Acceptance Criteria:
- [ ] New SharedCardBehavior.test.ts passes
- [ ] All 17 card test files updated to remove duplicate shared tests
- [ ] Total test count remains the same (tests moved, not removed)
- [ ] Runtime measurably reduced

---

### Stream 3: Reduce `nextTick()` in Filter Tests [MEDIUM EFFORT, MEDIUM IMPACT]

**Estimated time savings:** 1-2 seconds
**Effort:** 2-3 hours
**Parallelizable:** YES (each file independent)

#### Files to Refactor:

| File | `nextTick()` Count | Priority |
|------|-------------------|----------|
| `tests/pages/monsters/filters.test.ts` | 54 | HIGH |
| `tests/pages/items/filters.test.ts` | 39 | HIGH |
| `tests/pages/feats/filter-api.test.ts` | 32 | MEDIUM |
| `tests/pages/backgrounds/new-filters.test.ts` | 32 | MEDIUM |
| `tests/composables/usePageFilterSetup.test.ts` | 12 | LOW |

#### Pattern to Apply:

**Before (direct mutation + tick):**
```typescript
const component = wrapper.vm as any
component.selectedType = 'dragon'
await wrapper.vm.$nextTick()
expect(wrapper.text()).toContain('dragon')
```

**After (store mutation):**
```typescript
const store = useMonsterFiltersStore()
store.selectedType = 'dragon'
await wrapper.vm.$nextTick() // Often only 1 tick needed after store update
expect(wrapper.text()).toContain('dragon')
```

**After (batch mutations):**
```typescript
const store = useMonsterFiltersStore()
store.$patch({
  selectedType: 'dragon',
  selectedSize: 'large',
  minCR: 5
})
await wrapper.vm.$nextTick() // Single tick for all changes
```

#### Acceptance Criteria:
- [ ] `nextTick()` count reduced by 50%+ in each file
- [ ] All tests still pass
- [ ] No flaky tests introduced

---

### Stream 4: Split Large Test Files [MEDIUM EFFORT, LOW IMPACT]

**Estimated time savings:** 2-3 seconds (parallelization benefit)
**Effort:** 2-3 hours
**Parallelizable:** NO (sequential restructuring)

#### Files to Split:

**`tests/pages/monsters/filters.test.ts` (781 lines) → 4 files:**
- `tests/pages/monsters/filters.layout.test.ts` — Layout and structure tests
- `tests/pages/monsters/filters.cr.test.ts` — Challenge Rating filter tests
- `tests/pages/monsters/filters.basic.test.ts` — Size, Type, Alignment filters
- `tests/pages/monsters/filters.integration.test.ts` — Combined filter tests

**`tests/pages/classes/detail/reference.test.ts` (663 lines) → 3 files:**
- `tests/pages/classes/detail/reference.overview.test.ts`
- `tests/pages/classes/detail/reference.features.test.ts`
- `tests/pages/classes/detail/reference.progression.test.ts`

#### Acceptance Criteria:
- [ ] Large files split into focused, smaller files
- [ ] All tests still pass
- [ ] Improved test organization and maintainability

---

### Stream 5: Consolidate Reference Entity Card Tests [HIGH EFFORT, MEDIUM IMPACT]

**Estimated time savings:** 3-4 seconds
**Effort:** 3-4 hours
**Parallelizable:** NO (requires architectural decision)

#### Files to Consolidate:

10 "simple" reference entity cards have nearly identical test structure:
- `tests/components/ability-score/AbilityScoreCard.test.ts`
- `tests/components/condition/ConditionCard.test.ts`
- `tests/components/damage-type/DamageTypeCard.test.ts`
- `tests/components/item-type/ItemTypeCard.test.ts`
- `tests/components/language/LanguageCard.test.ts`
- `tests/components/proficiency-type/ProficiencyTypeCard.test.ts`
- `tests/components/size/SizeCard.test.ts`
- `tests/components/skill/SkillCard.test.ts`
- `tests/components/source/SourceCard.test.ts`
- `tests/components/spell-school/SpellSchoolCard.test.ts`

#### Approach Options:

**Option A: Parameterized Test File**
Create `tests/components/reference/ReferenceCards.test.ts` that tests all 10 cards with `describe.each()`.

**Option B: Shared Test Helper**
Create `testReferenceCard()` helper that each file calls with config.

**Recommendation:** Option A for maximum mount reduction.

#### Acceptance Criteria:
- [ ] Reference card tests consolidated
- [ ] Test count preserved
- [ ] Runtime reduced

---

### Stream 6: Quick Cleanup Tasks [LOW EFFORT, LOW IMPACT]

**Estimated time savings:** 0.5 seconds
**Effort:** 30 minutes
**Parallelizable:** YES

#### Tasks:

1. **Remove type-only tests** in `tests/composables/useEntityList.test.ts:11-28`
   - These verify TypeScript exports, which `npm run typecheck` handles

2. **Resolve skipped test** in `tests/pages/items/filters.test.ts`
   - Either implement or remove with documented reason

3. **Document fake timer usage** in `tests/composables/usePageFilterSetup.test.ts`
   - Add comment explaining why global fake timers are needed

#### Acceptance Criteria:
- [ ] Type-only tests removed
- [ ] Skipped test resolved or documented
- [ ] Fake timer usage documented

---

## Parallelization Strategy

### Independent Work Streams (Can Run Concurrently)

```
┌─────────────────────────────────────────────────────────────────┐
│                     PARALLEL EXECUTION                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Stream 1: it.each() Conversion                                │
│  ├── ItemCard.test.ts          (Agent 1)                       │
│  ├── RaceCard.test.ts          (Agent 2)                       │
│  ├── MonsterCard.test.ts       (Agent 3)                       │
│  ├── ClassCard.test.ts         (Agent 4)                       │
│  ├── DamageTypeCard.test.ts    (Agent 5)                       │
│  ├── LanguageCard.test.ts      (Agent 6)                       │
│  ├── SizeCard.test.ts          (Agent 7)                       │
│  └── SkillCard.test.ts         (Agent 8)                       │
│                                                                 │
│  Stream 3: nextTick() Reduction                                │
│  ├── monsters/filters.test.ts  (Agent 9)                       │
│  ├── items/filters.test.ts     (Agent 10)                      │
│  ├── feats/filter-api.test.ts  (Agent 11)                      │
│  └── backgrounds/new-filters.test.ts (Agent 12)                │
│                                                                 │
│  Stream 6: Quick Cleanup                                        │
│  └── All cleanup tasks         (Agent 13)                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Sequential Work Streams (Must Run in Order)

```
┌─────────────────────────────────────────────────────────────────┐
│                    SEQUENTIAL EXECUTION                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Stream 2: Card Test Consolidation                             │
│  ├── Phase 2A: Create SharedCardBehavior.test.ts (First)       │
│  └── Phase 2B: Update 17 card test files (After 2A)            │
│                                                                 │
│  Stream 4: Split Large Files                                    │
│  └── Requires careful restructuring                             │
│                                                                 │
│  Stream 5: Reference Card Consolidation                         │
│  └── Architectural decision needed first                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Suggested Session Breakdown

### Session 1: Quick Wins + it.each() Conversion
- Stream 6: Quick cleanup (30 min)
- Stream 1: Convert all loops to `it.each()` (2 hours)
- Run tests, measure improvement
- **Expected savings:** 5-10 seconds

### Session 2: Card Test Consolidation
- Stream 2A: Create SharedCardBehavior.test.ts (1.5 hours)
- Stream 2B: Update 17 card files (2 hours)
- Run tests, measure improvement
- **Expected savings:** 8-15 seconds

### Session 3: Filter Test Optimization
- Stream 3: Reduce `nextTick()` in filter tests (3 hours)
- Run tests, measure improvement
- **Expected savings:** 1-2 seconds

### Session 4: Structural Improvements
- Stream 4: Split large test files (2-3 hours)
- Stream 5: Consolidate reference cards (3-4 hours)
- Run tests, measure improvement
- **Expected savings:** 5-7 seconds

---

## Success Metrics

| Metric | Before | Target | Stretch |
|--------|--------|--------|---------|
| Full suite runtime | 125s | 85s | 75s |
| Improvement | — | 32% | 40% |
| `for` loops in tests | 8+ | 0 | 0 |
| `nextTick()` calls | 170+ | <80 | <50 |
| Redundant mounts | 100+ | <30 | <20 |

---

## Risk Mitigation

1. **Flaky tests after refactoring**
   - Run each modified test file 5x before committing
   - Use `vitest --reporter=verbose` to catch timing issues

2. **Test coverage reduction**
   - Track coverage before/after each stream
   - Ensure consolidation moves tests, doesn't remove them

3. **Merge conflicts**
   - Work on separate files in parallel
   - Coordinate on shared helpers

---

## Appendix: File Reference

### Card Component Test Files (17)
```
tests/components/ability-score/AbilityScoreCard.test.ts
tests/components/background/BackgroundCard.test.ts
tests/components/class/ClassCard.test.ts
tests/components/condition/ConditionCard.test.ts
tests/components/damage-type/DamageTypeCard.test.ts
tests/components/feat/FeatCard.test.ts
tests/components/item/ItemCard.test.ts
tests/components/item-type/ItemTypeCard.test.ts
tests/components/language/LanguageCard.test.ts
tests/components/monster/MonsterCard.test.ts
tests/components/proficiency-type/ProficiencyTypeCard.test.ts
tests/components/race/RaceCard.test.ts
tests/components/size/SizeCard.test.ts
tests/components/skill/SkillCard.test.ts
tests/components/source/SourceCard.test.ts
tests/components/spell/SpellCard.test.ts
tests/components/spell-school/SpellSchoolCard.test.ts
```

### Filter Store Test Files (7)
```
tests/stores/backgroundFilters.test.ts
tests/stores/classFilters.test.ts
tests/stores/featFilters.test.ts
tests/stores/itemFilters.test.ts
tests/stores/monsterFilters.test.ts
tests/stores/raceFilters.test.ts
tests/stores/spellFilters.test.ts
```

### Page Filter Test Files (5)
```
tests/pages/backgrounds/new-filters.test.ts
tests/pages/feats/filter-api.test.ts
tests/pages/items/filters.test.ts
tests/pages/monsters/filters.test.ts
tests/composables/usePageFilterSetup.test.ts
```
