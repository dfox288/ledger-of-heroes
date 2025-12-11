# Frontend Migration Plan: Slug Consolidation (#507)

**Date**: 2025-12-11
**Frontend Issue**: #507
**Backend Issue**: #506
**Status**: ✅ Code Changes Complete - Awaiting Backend Deployment
**Breaking Change**: Yes (API response format)

## Overview

The backend is consolidating `full_slug` and `slug` into a single `slug` field that contains source-prefixed values (e.g., `phb:fireball` instead of `fireball`).

### What's Changing

| Before | After |
|--------|-------|
| `slug: "fireball"` | `slug: "phb:fireball"` |
| `full_slug: "phb:fireball"` | _(removed)_ |

### Key Insight from Backend Plan

> "Character reference fields (`race_slug`, `background_slug`, etc.) unchanged - already stored prefixed values"

This means our API calls that send `race_slug`, `class_slug`, etc. are already sending prefixed values. The main work is updating code that **reads** `full_slug` from API responses.

---

## Impact Assessment

### High Impact (Core Logic)

| File | Lines | Description |
|------|-------|-------------|
| `app/stores/characterWizard.ts` | ~30 refs | Primary character creation store - uses `full_slug` extensively for API calls |
| `app/composables/useWizardChoiceSelection.ts` | 2 refs | Choice selection composable |

### Medium Impact (Components)

| File | Refs | Description |
|------|------|-------------|
| `app/pages/characters/[publicId]/level-up/index.vue` | 3 | Level-up page class selection |
| `app/components/character/wizard/StepSpells.vue` | 2 | Spell selection tracking |
| `app/components/character/wizard/StepLanguages.vue` | 1 | Language selection |
| `app/components/character/wizard/StepFeats.vue` | 1 | Feat selection |
| `app/components/character/wizard/EquipmentChoiceList.vue` | 1 | Equipment selection |
| `app/components/character/levelup/StepSubclassChoice.vue` | 1 | Subclass selection |
| `app/components/character/FeatCard.vue` | 1 | Feat prerequisite display |
| `app/components/feat/FeatCard.vue` | 1 | Feat prerequisite display |
| `app/components/character/picker/FeatDetailModal.vue` | 1 | Feat detail modal |

### Low Impact (Types & Tests)

| File | Description |
|------|-------------|
| `app/types/api/generated.ts` | Auto-regenerated via `sync-api-types.js` |
| `app/types/character.ts` | Manual type definitions |
| `tests/helpers/mockFactories.ts` | 25+ `full_slug` references |
| `tests/stores/characterWizard.test.ts` | 20+ `full_slug` references |
| `tests/stores/characterLevelUp.test.ts` | 4 `full_slug` references |
| `tests/components/character/wizard/*.test.ts` | Various mock data |
| `tests/components/character/sheet/*.test.ts` | Various mock data |
| `tests/composables/useWizardChoiceSelection.test.ts` | 10+ `full_slug` references |
| `scripts/character-stress-test.ts` | 10+ `full_slug` references |

---

## Migration Strategy

### Approach: Simple Find-Replace with Validation

Since `slug` will now contain the exact values that `full_slug` had, this is largely a renaming exercise:

```typescript
// BEFORE
const raceSlug = race.full_slug ?? race.slug

// AFTER
const raceSlug = race.slug
```

The fallback patterns (`full_slug ?? slug`) exist because the backend was inconsistent. After consolidation, `slug` always has the prefixed value - no fallback needed.

---

## Phase 1: Type Updates

### Task 1.1: Sync API Types (Auto)

After backend deploys, regenerate types:

```bash
# Run from HOST (not Docker) - requires backend running
node scripts/sync-api-types.js
```

This will remove `full_slug` from all generated types automatically.

### Task 1.2: Update Manual Types

**File**: `app/types/character.ts`

```typescript
// Lines 258, 264, 266 - Remove full_slug from embedded entity types
// BEFORE
race: { id: number, name: string, slug: string, full_slug: string } | null

// AFTER
race: { id: number, name: string, slug: string } | null
```

---

## Phase 2: Store Updates

### Task 2.1: Update characterWizard.ts

**File**: `app/stores/characterWizard.ts`

**Changes needed:**

1. **Line 41** - Remove `full_slug` from `SelectableEntity` interface:
   ```typescript
   // BEFORE
   full_slug: string

   // AFTER
   // Remove entirely - slug now contains prefixed value
   ```

2. **Lines 458-459, 513-514** - Simplify race slug access:
   ```typescript
   // BEFORE
   const raceSlug = race.full_slug ?? race.slug

   // AFTER
   const raceSlug = race.slug
   ```

3. **Lines 551-560** - Simplify class slug access:
   ```typescript
   // BEFORE
   const newClassSlug = cls.full_slug
   if (!newClassSlug) {
     throw new Error('Class missing full_slug - cannot save')
   }

   // AFTER
   const newClassSlug = cls.slug
   ```

4. **Lines 597-613** - Simplify subclass slug access:
   ```typescript
   // BEFORE
   if (!subclass.full_slug) {
     throw new Error('Subclass missing full_slug - cannot save')
   }

   // AFTER
   // slug is always present - validation not needed
   ```

5. **Lines 639-642** - Simplify background slug access:
   ```typescript
   // BEFORE
   const backgroundSlug = background.full_slug
   if (!backgroundSlug) {
     throw new Error('Background missing full_slug - cannot save')
   }

   // AFTER
   const backgroundSlug = background.slug
   ```

6. **Lines 773-776, 861** - Update type casts and response handling

---

## Phase 3: Component Updates

### Task 3.1: Update Wizard Step Components

**Pattern**: Replace `.full_slug` with `.slug` throughout:

| File | Change |
|------|--------|
| `StepSpells.vue` | `spell.full_slug` → `spell.slug` |
| `StepLanguages.vue` | `lang.full_slug` → `lang.slug` |
| `StepFeats.vue` | `feat.full_slug` → `feat.slug` |
| `EquipmentChoiceList.vue` | `item.full_slug` → `item.slug` |

### Task 3.2: Update Level-Up Components

| File | Change |
|------|--------|
| `level-up/index.vue` | `class.full_slug` → `class.slug` |
| `StepSubclassChoice.vue` | `subclass.full_slug` → `subclass.slug` |

### Task 3.3: Update Feat Cards

| File | Change |
|------|--------|
| `character/FeatCard.vue` | `prereq.race?.full_slug` → `prereq.race?.slug` |
| `feat/FeatCard.vue` | `prereq.race?.full_slug` → `prereq.race?.slug` |
| `picker/FeatDetailModal.vue` | Similar pattern |

---

## Phase 4: Composable Updates

### Task 4.1: Update useWizardChoiceSelection.ts

**File**: `app/composables/useWizardChoiceSelection.ts`

```typescript
// Line 349-350
// BEFORE
const slug = (opt.full_slug as string) ?? (opt.slug as string)

// AFTER
const slug = opt.slug as string
```

---

## Phase 5: Test Updates

### Task 5.1: Update Mock Factories

**File**: `tests/helpers/mockFactories.ts`

Remove `full_slug` property from all mock factory functions. The `slug` property now contains the prefixed value.

```typescript
// BEFORE
export function createMockSpell(overrides = {}) {
  return {
    slug: 'fireball',
    full_slug: 'phb:fireball',
    // ...
  }
}

// AFTER
export function createMockSpell(overrides = {}) {
  return {
    slug: 'phb:fireball',
    // ...
  }
}
```

**Factories to update**: ~15 functions

### Task 5.2: Update Test Files

Update all test files that reference `full_slug` in mock data:

| Test File | Changes |
|-----------|---------|
| `tests/stores/characterWizard.test.ts` | ~20 references |
| `tests/stores/characterLevelUp.test.ts` | ~4 references |
| `tests/components/character/wizard/StepSpells.test.ts` | ~5 references |
| `tests/components/character/wizard/StepFeats.test.ts` | ~5 references |
| `tests/components/character/wizard/StepEquipment.test.ts` | ~10 references |
| `tests/components/character/wizard/StepProficiencies.test.ts` | ~1 reference |
| `tests/components/character/wizard/StepSize.test.ts` | ~1 reference |
| `tests/components/character/sheet/SpellsByLevel.test.ts` | ~25 references |
| `tests/composables/useWizardChoiceSelection.test.ts` | ~10 references |

---

## Phase 6: Scripts & Utilities

### Task 6.1: Update Stress Test Script

**File**: `scripts/character-stress-test.ts`

Update all `full_slug` references:
- Line 271, 276, 330, 335, 353, 376, 398, 408, 429, 437

```typescript
// BEFORE
.map(o => o.full_slug || o.slug)

// AFTER
.map(o => o.slug)
```

---

## Phase 7: Verification

### Task 7.1: Run Test Suite

```bash
docker compose exec nuxt npm run test
```

### Task 7.2: TypeScript Check

```bash
docker compose exec nuxt npm run typecheck
```

### Task 7.3: Run Character Stress Test

```bash
npm run test:character-stress -- --count=10 --cleanup
```

### Task 7.4: Manual Verification

1. Create a new character through the wizard
2. Complete all choice steps
3. Level up a character
4. Verify character sheet displays correctly

---

## Migration Checklist

### Pre-Migration

- [ ] Backend PR for #506 is merged
- [ ] Backend is deployed and running
- [ ] Create feature branch: `feature/issue-506-slug-consolidation-frontend`

### Phase 1: Types

- [ ] Run `node scripts/sync-api-types.js`
- [ ] Update `app/types/character.ts`
- [ ] Run `npm run typecheck` (expect failures)

### Phase 2: Store

- [ ] Update `app/stores/characterWizard.ts`
- [ ] Remove `full_slug` from `SelectableEntity` interface
- [ ] Simplify all `full_slug ?? slug` patterns

### Phase 3: Components

- [ ] Update `StepSpells.vue`
- [ ] Update `StepLanguages.vue`
- [ ] Update `StepFeats.vue`
- [ ] Update `EquipmentChoiceList.vue`
- [ ] Update `level-up/index.vue`
- [ ] Update `StepSubclassChoice.vue`
- [ ] Update `character/FeatCard.vue`
- [ ] Update `feat/FeatCard.vue`
- [ ] Update `picker/FeatDetailModal.vue`

### Phase 4: Composables

- [ ] Update `useWizardChoiceSelection.ts`

### Phase 5: Tests

- [ ] Update `tests/helpers/mockFactories.ts`
- [ ] Update all test files with `full_slug` references
- [ ] Run `npm run test` (all should pass)

### Phase 6: Scripts

- [ ] Update `scripts/character-stress-test.ts`

### Phase 7: Verification

- [ ] `npm run typecheck` passes
- [ ] `npm run test` passes
- [ ] `npm run lint` passes
- [ ] Manual testing passes
- [ ] Stress test passes

### Post-Migration

- [ ] Commit changes
- [ ] Create PR referencing #506
- [ ] Update CHANGELOG.md

---

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Missed `full_slug` reference | Low | Medium | Grep search + TypeScript errors will catch |
| API mismatch during transition | Medium | High | Coordinate timing with backend deployment |
| Test data out of sync | Low | Low | Run full test suite after changes |

---

## Timeline Estimate

| Phase | Estimate |
|-------|----------|
| Types | 15 min |
| Store | 30 min |
| Components | 30 min |
| Composables | 10 min |
| Tests | 45 min |
| Scripts | 10 min |
| Verification | 30 min |
| **Total** | **~3 hours** |

---

## Notes

1. **No URL changes needed** - Entity detail pages use `slug` for URLs, which will now be prefixed (`/spells/phb:fireball`). This is intentional and matches the backend's design.

2. **Meilisearch filters unchanged** - We already send prefixed values in filters because we were using `full_slug`.

3. **Character saves unchanged** - The character API already stores prefixed values in `race_slug`, `class_slug`, etc.
