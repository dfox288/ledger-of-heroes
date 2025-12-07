# Test Suite Audit & Domain-Specific Restructuring Proposal

**Date:** 2025-11-26
**Current State:** 116 test files, 1515 tests, ~125-160s runtime
**Goal:** Enable running domain-specific test subsets for faster feedback during feature development

---

## Current Test Suite Overview

### File Distribution

| Category | Files | Description |
|----------|-------|-------------|
| **components/** | 65 | Component unit tests |
| **pages/** | 28 | Page-level integration tests (filter tests) |
| **composables/** | 11 | Composable unit tests |
| **stores/** | 7 | Pinia store tests (filter persistence) |
| **utils/** | 2 | Utility function tests |
| **server/** | 1 | Server API route tests |
| **features/** | 1 | Feature integration tests |
| **types/** | 1 | Type validation tests |
| **e2e/** | 2 | Playwright E2E tests (excluded from unit tests) |
| **Total** | **118** | (116 unit + 2 E2E) |

### Component Tests Breakdown

#### Entity Card Components (17 files)
| Entity | Files | Notes |
|--------|-------|-------|
| Main Entities (7) | 7 | spell, item, race, class, background, feat, monster |
| Reference Entities (10) | 10 | ability-score, condition, damage-type, item-type, language, proficiency-type, size, skill, spell-school, source |

#### UI Components (47 files)
| Category | Files | Notes |
|----------|-------|-------|
| ui/accordion/ | 17 | Accordion slot components |
| ui/detail/ | 8 | Detail page components |
| ui/filter/ | 5 | Filter UI components |
| ui/list/ | 6 | List page components |
| ui/class/ | 4 | Class-specific components |
| ui/card/ | 1 | Card components |
| ui/other | 6 | Misc UI (JsonDebugPanel, BackLink, etc.) |

### Page Tests Breakdown (28 files)

**Problem:** Page tests are not well-organized. Many filter tests are at root level instead of in entity subdirectories.

| Entity | Files | Current Location |
|--------|-------|------------------|
| spells | 5 | `pages/spells/` (3) + `pages/spells-*.test.ts` (2) |
| items | 5 | `pages/items/` (1) + `pages/items-*.test.ts` (4) |
| races | 4 | `pages/races/` (2) + `pages/races-*.test.ts` (2) |
| classes | 3 | `pages/classes/` (1) + `pages/classes-*.test.ts` (2) |
| backgrounds | 4 | `pages/backgrounds/` (1) + `pages/backgrounds-*.test.ts` (3) |
| feats | 3 | `pages/feats/` (1) + `pages/feats-*.test.ts` (2) |
| monsters | 4 | `pages/monsters/` (1) + `pages/monsters-*.test.ts` (3) |

---

## Proposed Domain Structure

### Domain Categories

| Domain | Description | Test Patterns |
|--------|-------------|---------------|
| `core` | Shared infrastructure (composables, utils, stores base) | Foundation code used by all features |
| `ui` | Reusable UI components | Shared across all entities |
| `spells` | Spells entity | Card, page, filters, stores |
| `items` | Items entity | Card, page, filters, stores |
| `races` | Races entity | Card, page, filters, stores |
| `classes` | Classes entity | Card, page, filters, stores |
| `backgrounds` | Backgrounds entity | Card, page, filters, stores |
| `feats` | Feats entity | Card, page, filters, stores |
| `monsters` | Monsters entity | Card, page, filters, stores |
| `reference` | Reference entities (10 types) | Cards for ability-scores, conditions, etc. |

### Proposed npm Scripts

```json
{
  "scripts": {
    "test": "vitest",
    "test:watch": "vitest watch",

    "test:core": "vitest --project core",
    "test:ui": "vitest --project ui",

    "test:spells": "vitest --project spells",
    "test:items": "vitest --project items",
    "test:races": "vitest --project races",
    "test:classes": "vitest --project classes",
    "test:backgrounds": "vitest --project backgrounds",
    "test:feats": "vitest --project feats",
    "test:monsters": "vitest --project monsters",
    "test:reference": "vitest --project reference",

    "test:e2e": "playwright test"
  }
}
```

### Vitest Workspace Configuration

Using Vitest's workspace feature, we can define domain-specific projects:

**`vitest.workspace.ts`** (new file):
```typescript
import { defineWorkspace } from 'vitest/config'

export default defineWorkspace([
  // Core: Composables, utils, shared stores
  {
    extends: './vitest.config.ts',
    test: {
      name: 'core',
      include: [
        'tests/composables/**/*.test.ts',
        'tests/utils/**/*.test.ts',
        'tests/server/**/*.test.ts',
        'tests/types/**/*.test.ts',
      ],
    },
  },

  // UI: Shared UI components
  {
    extends: './vitest.config.ts',
    test: {
      name: 'ui',
      include: [
        'tests/components/ui/**/*.test.ts',
        'tests/components/pages/**/*.test.ts',
      ],
    },
  },

  // Spells domain
  {
    extends: './vitest.config.ts',
    test: {
      name: 'spells',
      include: [
        'tests/components/spell/**/*.test.ts',
        'tests/components/spell-school/**/*.test.ts',
        'tests/pages/spells/**/*.test.ts',
        'tests/pages/spells-*.test.ts',
        'tests/stores/spellFilters.test.ts',
        'tests/features/spell-*.test.ts',
      ],
    },
  },

  // Items domain
  {
    extends: './vitest.config.ts',
    test: {
      name: 'items',
      include: [
        'tests/components/item/**/*.test.ts',
        'tests/components/item-type/**/*.test.ts',
        'tests/pages/items/**/*.test.ts',
        'tests/pages/items-*.test.ts',
        'tests/stores/itemFilters.test.ts',
      ],
    },
  },

  // Races domain
  {
    extends: './vitest.config.ts',
    test: {
      name: 'races',
      include: [
        'tests/components/race/**/*.test.ts',
        'tests/components/size/**/*.test.ts',
        'tests/pages/races/**/*.test.ts',
        'tests/pages/races-*.test.ts',
        'tests/stores/raceFilters.test.ts',
      ],
    },
  },

  // Classes domain
  {
    extends: './vitest.config.ts',
    test: {
      name: 'classes',
      include: [
        'tests/components/class/**/*.test.ts',
        'tests/components/ui/class/**/*.test.ts',
        'tests/pages/classes/**/*.test.ts',
        'tests/pages/classes-*.test.ts',
        'tests/stores/classFilters.test.ts',
      ],
    },
  },

  // Backgrounds domain
  {
    extends: './vitest.config.ts',
    test: {
      name: 'backgrounds',
      include: [
        'tests/components/background/**/*.test.ts',
        'tests/pages/backgrounds/**/*.test.ts',
        'tests/pages/backgrounds-*.test.ts',
        'tests/stores/backgroundFilters.test.ts',
      ],
    },
  },

  // Feats domain
  {
    extends: './vitest.config.ts',
    test: {
      name: 'feats',
      include: [
        'tests/components/feat/**/*.test.ts',
        'tests/pages/feats/**/*.test.ts',
        'tests/pages/feats-*.test.ts',
        'tests/stores/featFilters.test.ts',
      ],
    },
  },

  // Monsters domain
  {
    extends: './vitest.config.ts',
    test: {
      name: 'monsters',
      include: [
        'tests/components/monster/**/*.test.ts',
        'tests/pages/monsters/**/*.test.ts',
        'tests/pages/monsters-*.test.ts',
        'tests/stores/monsterFilters.test.ts',
      ],
    },
  },

  // Reference entities (lookups)
  {
    extends: './vitest.config.ts',
    test: {
      name: 'reference',
      include: [
        'tests/components/ability-score/**/*.test.ts',
        'tests/components/condition/**/*.test.ts',
        'tests/components/damage-type/**/*.test.ts',
        'tests/components/language/**/*.test.ts',
        'tests/components/proficiency-type/**/*.test.ts',
        'tests/components/skill/**/*.test.ts',
        'tests/components/source/**/*.test.ts',
      ],
    },
  },
])
```

---

## Expected Time Savings

| Suite | Est. Files | Est. Tests | Est. Runtime |
|-------|-----------|------------|--------------|
| Full Suite | 116 | 1515 | ~125-160s |
| `core` | 15 | ~120 | ~15s |
| `ui` | 48 | ~400 | ~45s |
| `spells` | 8 | ~60 | ~10s |
| `items` | 7 | ~80 | ~12s |
| `races` | 5 | ~55 | ~8s |
| `classes` | 6 | ~55 | ~9s |
| `backgrounds` | 5 | ~75 | ~10s |
| `feats` | 4 | ~50 | ~7s |
| `monsters` | 5 | ~85 | ~12s |
| `reference` | 7 | ~60 | ~8s |

**Key Benefit:** When working on spells filters, run `npm run test:spells` (~10s) instead of full suite (~125s).

---

## Recommended File Reorganization

### Phase 1: Move Root-Level Page Tests (Low Risk)

Move scattered page filter tests into their entity subdirectories:

```
# Spells
tests/pages/spells-level-filter.test.ts → tests/pages/spells/level-filter.test.ts
tests/pages/spells-tag-filter.test.ts → tests/pages/spells/tag-filter.test.ts

# Items
tests/pages/items-filters.test.ts → tests/pages/items/filters.test.ts
tests/pages/items-api-filters.test.ts → tests/pages/items/api-filters.test.ts
tests/pages/items-filters-advanced.test.ts → tests/pages/items/filters-advanced.test.ts
tests/pages/items-filters-weapon-armor.test.ts → tests/pages/items/filters-weapon-armor.test.ts

# Races
tests/pages/races-filters.test.ts → tests/pages/races/filters.test.ts
tests/pages/races-parent-filter.test.ts → tests/pages/races/parent-filter.test.ts

# Classes
tests/pages/classes-filters.test.ts → tests/pages/classes/filters.test.ts
tests/pages/classes-filters-expanded.test.ts → tests/pages/classes/filters-expanded.test.ts

# Backgrounds
tests/pages/backgrounds-filters.test.ts → tests/pages/backgrounds/filters.test.ts
tests/pages/backgrounds-new-filters.test.ts → tests/pages/backgrounds/new-filters.test.ts
tests/pages/backgrounds-source-filter.test.ts → tests/pages/backgrounds/source-filter.test.ts

# Feats
tests/pages/feats-filters.test.ts → tests/pages/feats/filters.test.ts
tests/pages/feats-filter-api.test.ts → tests/pages/feats/filter-api.test.ts

# Monsters
tests/pages/monsters-filters.test.ts → tests/pages/monsters/filters.test.ts
tests/pages/monsters-api-filters.test.ts → tests/pages/monsters/api-filters.test.ts
tests/pages/monsters-filter-chips.test.ts → tests/pages/monsters/filter-chips.test.ts
```

**Why:** Cleaner structure + simpler glob patterns for workspace config.

---

## Implementation Steps

1. **Create `vitest.workspace.ts`** - Define domain projects
2. **Update `package.json`** - Add domain-specific scripts
3. **Reorganize page tests** (optional) - Move to subdirectories
4. **Test workspace config** - Verify each domain runs correctly
5. **Update CLAUDE.md** - Document new test commands

---

## Alternative: Simple Include Patterns (No Workspace)

If workspace setup is too complex, use simple CLI patterns:

```bash
# Run spells tests only
npm run test -- "tests/**/*spell*"

# Run items tests only
npm run test -- "tests/**/*item*"

# Run UI tests only
npm run test -- "tests/components/ui/**"
```

**Pros:** No config changes needed
**Cons:** Less reliable pattern matching, no reusable scripts

---

## Recommendation

**Start with Vitest Workspace approach** because:
1. Explicit file inclusion (no accidental misses)
2. Reusable npm scripts
3. Can extend base config (shared setup, environment)
4. Better IDE support for running specific projects
5. Future-proof for larger test suites

**Optional:** Reorganize page tests after workspace is working.
