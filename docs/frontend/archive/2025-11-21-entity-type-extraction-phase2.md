# Phase 2: Entity Type Extraction - Design Document

**Date:** 2025-11-21
**Phase:** 2 of 3 (Type System Evolution)
**Status:** Ready for Implementation
**Complexity:** Low-Medium (well-understood pattern from Phase 1)

---

## Overview

Extract 6 entity type definitions (Spell, Item, Race, CharacterClass, Background, Feat) from inline card component interfaces into centralized type system at `app/types/api/entities.ts`.

**Goals:**
1. Eliminate duplicate interface definitions in test files
2. Enable type-safe test data creation
3. Prepare foundation for future OpenAPI type generation
4. Maintain 100% test coverage (525/525 passing)

---

## Context

### Phase 1 Results (Completed Earlier Today)
- Created `app/types/api/common.ts` with 4 shared types
- Migrated 12 components to use centralized types
- Eliminated 15+ duplicate definitions
- All tests passing, zero regressions

### Phase 2 Scope
Extract entity-specific interfaces that are currently:
- Defined once per card component (6 total)
- Duplicated in test files (6 test files × ~15 lines each = 90 duplicate lines)
- Not reusable across components or composables

---

## Problem Statement

### Current State
Each entity card component defines its interface inline:

```typescript
// SpellCard.vue
interface Spell {
  id: number
  name: string
  slug: string
  level: number
  school?: { id: number; code: string; name: string }
  casting_time: string
  range: string
  description: string
  is_ritual: boolean
  needs_concentration: boolean
  sources?: Source[]
}
```

**Issues:**
1. **Test duplication** - Test files recreate entire interface for mock data
2. **No type safety** - Mock data not validated against component interface
3. **Maintenance burden** - API changes require updating component + test
4. **Can't reuse** - Detail pages, composables can't import entity types

### Root Cause
No centralized entity type definitions. Phase 1 solved this for shared types (Source, Tag, etc.), but entity types remain inline.

---

## Proposed Solution

### Architecture

**File Structure:**
```
app/types/
├── api/
│   ├── common.ts      # Shared primitives (Source, Tag, etc.) ✅ Phase 1
│   ├── entities.ts    # Entity models (Spell, Item, etc.) ⬅️ NEW
│   └── responses.ts   # API wrappers (future)
└── index.ts           # Barrel exports (updated)
```

**Why separate files?**
- `common.ts` = Primitives/Value Objects (used across entities)
- `entities.ts` = Domain models (core business objects)
- Mirrors backend architecture (Value Objects vs Entities)
- Prepares for OpenAPI generation (likely generates separate files)

---

## Design Decisions

### 1. Conservative Extraction (No Enhancements)

**Principle:** Extract interfaces **exactly as-is** from current components.

**Example:**
```typescript
// Current: RaceCard.vue
interface Race {
  modifiers?: any[]   // Untyped array
  traits?: any[]      // Untyped array
}

// Extracted: entities.ts (SAME)
export interface Race {
  modifiers?: any[]   // Keep as-is, add TODO comment
  traits?: any[]      // Keep as-is, add TODO comment
}
```

**Why:**
- Zero behavior changes = tests stay green
- Avoid scope creep (typing nested arrays is separate task)
- Easy to review (pure extraction, no logic changes)

**Future improvement:** Can type nested arrays later (separate PR)

---

### 2. Barrel Export Pattern

**Update:** `app/types/index.ts`

```typescript
// Phase 1 exports
export type { Source, AbilityScore, Modifier, Tag } from './api/common'

// Phase 2 exports (NEW)
export type {
  Spell,
  Item,
  Race,
  CharacterClass,
  Background,
  Feat
} from './api/entities'
```

**Import pattern (consistent):**
```typescript
import type { Spell, Source } from '~/types'  // Single import point
```

---

### 3. Test File Updates

**Before:**
```typescript
import SpellCard from '~/components/spell/SpellCard.vue'

describe('SpellCard', () => {
  const mockSpell = {  // Untyped object
    id: 1,
    name: 'Fireball',
    // ... no compile-time validation
  }
})
```

**After:**
```typescript
import type { Spell } from '~/types'
import SpellCard from '~/components/spell/SpellCard.vue'

describe('SpellCard', () => {
  const mockSpell: Spell = {  // Typed object
    id: 1,
    name: 'Fireball',
    // ... TypeScript validates this matches Spell interface
  }
})
```

**Benefits:**
- Type errors caught at compile time (missing fields, wrong types)
- IntelliSense in test data (auto-complete field names)
- Refactoring safety (rename fields, find all usages)

---

## Implementation Plan

### Approach: Incremental with Test Checkpoints

Following Phase 1's proven pattern:
- Create types file (no breaking changes)
- Migrate 2 entities per batch
- Test checkpoint after each batch
- Commit when green

**Why this approach:**
- ✅ Proven in Phase 1 (6 commits, zero regressions)
- ✅ Easy to debug (small changes per batch)
- ✅ Easy to rollback (clear checkpoints)
- ✅ Matches project's TDD culture

---

### Batch 1: Spell + Item

**Files created:**
- `app/types/api/entities.ts` (with Spell and Item)

**Files updated:**
- `app/types/index.ts` (add entity exports)
- `app/components/spell/SpellCard.vue` (import Spell from ~/types)
- `app/components/item/ItemCard.vue` (import Item from ~/types)
- `tests/components/spell/SpellCard.test.ts` (type mock data)
- `tests/components/item/ItemCard.test.ts` (type mock data)

**Validation:**
```bash
docker compose exec nuxt npm test -- --run
# Expected: 525/525 passing
```

**Commit message:**
```
refactor: Extract Spell and Item entity types

- Create app/types/api/entities.ts with Spell and Item interfaces
- Update SpellCard and ItemCard to import from centralized types
- Add type annotations to test mock data
- All 525 tests passing

Part of Phase 2: Entity Type Extraction
```

---

### Batch 2: Race + CharacterClass

**Files updated:**
- `app/types/api/entities.ts` (add Race and CharacterClass)
- `app/types/index.ts` (export new types)
- `app/components/race/RaceCard.vue`
- `app/components/class/ClassCard.vue`
- `tests/components/race/RaceCard.test.ts`
- `tests/components/class/ClassCard.test.ts`

**Validation:** Run tests → commit if green

---

### Batch 3: Background + Feat

**Files updated:**
- `app/types/api/entities.ts` (add Background and Feat)
- `app/types/index.ts` (export new types)
- `app/components/background/BackgroundCard.vue`
- `app/components/feat/FeatCard.vue`
- `tests/components/background/BackgroundCard.test.ts`
- `tests/components/feat/FeatCard.test.ts`

**Validation:** Run tests → commit if green

---

## Migration Pattern (Per Component)

### Card Component Update

**Before:**
```typescript
<script setup lang="ts">
import type { Source } from '~/types'

interface Spell {
  id: number
  name: string
  // ... 15 lines
  sources?: Source[]
}

interface Props {
  spell: Spell
}

defineProps<Props>()
</script>
```

**After:**
```typescript
<script setup lang="ts">
import type { Spell } from '~/types'

interface Props {
  spell: Spell
}

defineProps<Props>()
</script>
```

**Changes:**
- Import `Spell` instead of `Source` (or in addition to)
- Remove inline `Spell` interface
- Keep `Props` interface (component-specific)
- Zero template changes

**Lines saved:** ~15 per component

---

### Test File Update

**Before:**
```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import SpellCard from '~/components/spell/SpellCard.vue'

describe('SpellCard', () => {
  const mockSpell = {
    id: 1,
    name: 'Fireball',
    slug: 'fireball',
    level: 3,
    // ...
  }

  it('renders spell name', async () => {
    const wrapper = await mountSuspended(SpellCard, {
      props: { spell: mockSpell }
    })
    expect(wrapper.text()).toContain('Fireball')
  })
})
```

**After:**
```typescript
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import type { Spell } from '~/types'
import SpellCard from '~/components/spell/SpellCard.vue'

describe('SpellCard', () => {
  const mockSpell: Spell = {  // ⬅️ Type annotation added
    id: 1,
    name: 'Fireball',
    slug: 'fireball',
    level: 3,
    // ... TypeScript validates this matches Spell interface
  }

  it('renders spell name', async () => {
    const wrapper = await mountSuspended(SpellCard, {
      props: { spell: mockSpell }
    })
    expect(wrapper.text()).toContain('Fireball')
  })
})
```

**Changes:**
- Add import: `import type { Spell } from '~/types'`
- Add type annotation: `const mockSpell: Spell = {`
- No changes to test logic or assertions

---

## Entity Type Definitions

### Complete Interface Specifications

```typescript
import type { Source, AbilityScore, Modifier, Tag } from './common'

/**
 * Spell entity from D&D 5e API
 *
 * Used in: SpellCard, spell detail pages, tests
 * API endpoint: /api/v1/spells
 */
export interface Spell {
  id: number
  name: string
  slug: string
  level: number
  school?: {
    id: number
    code: string
    name: string
  }
  casting_time: string
  range: string
  description: string
  is_ritual: boolean
  needs_concentration: boolean
  sources?: Source[]
}

/**
 * Item entity from D&D 5e API
 *
 * Used in: ItemCard, item detail pages, tests
 * API endpoint: /api/v1/items
 */
export interface Item {
  id: number
  name: string
  slug: string
  rarity: string
  item_type?: {
    id: number
    name: string
  }
  is_magic: boolean
  requires_attunement: boolean
  cost_cp?: number
  weight?: number
  description?: string
  sources?: Source[]
}

/**
 * Race entity from D&D 5e API
 *
 * Used in: RaceCard, race detail pages, tests
 * API endpoint: /api/v1/races
 */
export interface Race {
  id: number
  name: string
  slug: string
  size?: {
    id: number
    name: string
    code: string
  }
  speed: number
  parent_race_id?: number | null
  parent_race?: {
    id: number
    slug: string
    name: string
    speed: number
  } | null
  subraces?: Array<{
    id: number
    slug: string
    name: string
  }>
  modifiers?: any[]  // TODO: Type with Modifier interface
  traits?: any[]     // TODO: Type trait structure
  description?: string
  sources?: Source[]
}

/**
 * Character Class entity from D&D 5e API
 *
 * Used in: ClassCard, class detail pages, tests
 * API endpoint: /api/v1/classes
 *
 * Note: Named CharacterClass to avoid conflict with JS 'class' keyword
 */
export interface CharacterClass {
  id: number
  name: string
  slug: string
  hit_die: number
  is_base_class: boolean
  parent_class_id?: number | null
  primary_ability?: {
    id: number
    code: string
    name: string
  } | null
  spellcasting_ability?: {
    id: number
    code: string
    name: string
  } | null
  subclasses?: any[]      // TODO: Type subclass structure
  proficiencies?: any[]   // TODO: Type proficiency structure
  description?: string
  sources?: Source[]
}

/**
 * Background entity from D&D 5e API
 *
 * Used in: BackgroundCard, background detail pages, tests
 * API endpoint: /api/v1/backgrounds
 */
export interface Background {
  id: number
  name: string
  slug: string
  skill_proficiencies?: any[]   // TODO: Type skill proficiency structure
  tool_proficiencies?: any[]    // TODO: Type tool proficiency structure
  languages?: any[]             // TODO: Type language structure
  feature_name?: string
  description?: string
  sources?: Source[]
}

/**
 * Feat entity from D&D 5e API
 *
 * Used in: FeatCard, feat detail pages, tests
 * API endpoint: /api/v1/feats
 */
export interface Feat {
  id: number
  name: string
  slug: string
  prerequisites?: any[]   // TODO: Type prerequisite structure
  modifiers?: any[]       // TODO: Use Modifier interface
  description?: string
  sources?: Source[]
}
```

---

## Impact Analysis

### Code Quality Improvements

**Before Phase 2:**
- 6 entity interfaces defined inline in components
- 6 test files with untyped mock data
- ~90 lines of duplicate interface definitions (tests)
- No compile-time validation of test data

**After Phase 2:**
- 6 entity interfaces in centralized location
- 6 test files with typed mock data
- ~90 duplicate lines removed
- Full TypeScript validation in tests

### Benefits

**Immediate:**
- ✅ Type-safe test data (catch errors at compile time)
- ✅ Better IntelliSense (autocomplete in tests)
- ✅ Single source of truth (change once, applies everywhere)
- ✅ Easier maintenance (API changes update one file)

**Future:**
- ✅ Ready for OpenAPI type generation
- ✅ Composables can import entity types
- ✅ Detail pages can use explicit types (currently inferred)
- ✅ Cross-component type sharing

### Metrics

| Metric | Before | After | Change |
|--------|--------|-------|--------|
| Entity type definitions | 6 (inline) | 6 (centralized) | Same types, better location |
| Test file duplicates | 6 × ~15 lines | 0 | -90 lines |
| Type safety in tests | None | Full | +6 files |
| Import statements | N/A | +12 | Small overhead |
| Net lines of code | Baseline | -~70 | Reduction |

---

## Risk Assessment

### Risks & Mitigation

**Risk 1: Breaking existing tests**
- **Likelihood:** Low (pure refactoring, no logic changes)
- **Mitigation:** Test checkpoint after each batch (2 entities at a time)
- **Impact if occurs:** Low (rollback to previous batch)

**Risk 2: TypeScript compilation errors**
- **Likelihood:** Low (extracting exact interfaces)
- **Mitigation:** Follow conservative extraction (no enhancements)
- **Impact if occurs:** Low (fix type definition in entities.ts)

**Risk 3: Mock data doesn't match interface**
- **Likelihood:** Medium (some test mocks may be incomplete)
- **Mitigation:** This is actually GOOD - we'll discover incomplete mocks
- **Impact if occurs:** Low (add missing fields to mock data)

---

## Success Criteria

### Definition of Done

Phase 2 is complete when:

- [x] `app/types/api/entities.ts` created with 6 entity interfaces
- [x] `app/types/index.ts` updated with entity exports
- [x] All 6 card components import from centralized types
- [x] All 6 test files use typed mock data
- [x] All 525 tests passing (zero regressions)
- [x] 3-4 clean commits with clear messages
- [x] CHANGELOG.md updated
- [x] Documentation updated (this design doc + handover)

### Quality Gates

**After each batch:**
- ✅ TypeScript compiles with no errors
- ✅ Full test suite passes (525/525)
- ✅ No linting warnings
- ✅ Component functionality unchanged (manual spot check)

**Final validation:**
```bash
# TypeScript check
docker compose exec nuxt npx nuxi typecheck

# Full test suite
docker compose exec nuxt npm test -- --run

# Lint check
docker compose exec nuxt npm run lint

# All should pass with zero errors
```

---

## Future Work

### Phase 3: OpenAPI Type Generation (Shelved)

**Status:** User requested to shelve for later

**When ready to implement:**

1. **Install type generator:**
   ```bash
   npm install -D openapi-typescript
   ```

2. **Generate types from backend API:**
   ```bash
   npx openapi-typescript http://localhost:8080/docs/api.json \
     --output app/types/api/generated.ts
   ```

3. **Replace manual types:**
   - Compare generated types with manual types in `entities.ts`
   - Update imports if structure differs
   - Remove manual definitions, use generated types

4. **Add to build process:**
   ```json
   // package.json
   {
     "scripts": {
       "types:generate": "openapi-typescript http://localhost:8080/docs/api.json -o app/types/api/generated.ts",
       "dev": "npm run types:generate && nuxt dev"
     }
   }
   ```

**Prerequisites:**
- ✅ Phase 1 complete (common types centralized)
- ✅ Phase 2 complete (entity types centralized) ⬅️ THIS PHASE
- Backend OpenAPI spec stable and accurate

**Estimated effort:** 2-3 hours

**Benefits:**
- Automatic type updates when API changes
- Guaranteed type accuracy (generated from source)
- No manual interface maintenance

**Note:** Phase 2 establishes the file structure and import patterns that OpenAPI generation will slot into seamlessly.

---

## Timeline Estimate

**Total estimated time:** 1.5-2 hours

**Breakdown:**
- Batch 1 (Spell + Item): 30-40 min
  - Create entities.ts: 10 min
  - Update components: 10 min
  - Update tests: 10 min
  - Testing: 5 min
  - Commit: 5 min

- Batch 2 (Race + Class): 25-30 min
  - Add to entities.ts: 10 min
  - Update components: 10 min
  - Update tests: 5 min
  - Testing: 5 min

- Batch 3 (Background + Feat): 25-30 min
  - Add to entities.ts: 10 min
  - Update components: 10 min
  - Update tests: 5 min
  - Testing: 5 min

- Documentation: 10-15 min
  - Update CHANGELOG.md
  - Update analysis doc with completion notes
  - Create handover doc

**Confidence level:** High (matches Phase 1 pattern exactly)

---

## Related Documents

**Phase 1 Documentation:**
- `docs/HANDOVER-2025-11-21-INTERFACE-EXTRACTION.md` - Phase 1 complete session
- `docs/plans/2025-11-21-interface-extraction-analysis.md` - Original analysis

**Current Status:**
- `docs/CURRENT_STATUS.md` - Overall project status

**This Document:**
- `docs/plans/2025-11-21-entity-type-extraction-phase2.md` - Phase 2 design (YOU ARE HERE)

---

**Design Status:** ✅ Ready for Implementation
**Next Step:** Set up worktree and begin Batch 1 (Spell + Item)
