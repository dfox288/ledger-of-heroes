# Ability Bonuses Endpoint Integration

**Date:** 2025-12-09
**Status:** Approved
**Issue:** #403

## Overview

Integrate the new `/characters/{id}/ability-bonuses` endpoint into `StepAbilities.vue` to replace the current N+1 feat-fetching pattern.

## Goals

| Goal | How |
|------|-----|
| Reduce API calls | Replace 1+N calls (race + each feat) with 1 call |
| Simplify code | Remove ~45 lines of feat-fetching logic |
| Improve consistency | Backend is single source of truth for bonuses |

## Scope

### In Scope

- Nitro proxy route for the new endpoint
- Type sync from backend
- Replace feat-fetching logic in `StepAbilities.vue`
- Update tests to use new mock shape

### Out of Scope

- Character sheet changes (stays as final scores only)
- New UI for bonus breakdown tooltips
- Changes to the choice selection flow (pending choices API)
- Handling future `source_type` values (`class_feature`, `item`, `condition`)

## Design

### Nitro Proxy Route

**File:** `server/api/characters/[id]/ability-bonuses.get.ts`

Standard proxy route following existing patterns in `server/api/characters/`.

### StepAbilities.vue Changes

**Removed (~45 lines):**
- `selectedFeats` ref - Map storing fetched feat data
- `featsPending` ref - Loading state for feat fetches
- Feat-fetching watch - Async logic that fetches each feat
- `featAbilityModifiers` computed - Extracts ability bonuses from fetched feats

**Added:**
- `abilityBonuses` fetch - Single call to `/characters/{id}/ability-bonuses`
- `raceBonuses` computed - Filter endpoint response where `source_type === 'race'`
- `featBonuses` computed - Filter endpoint response where `source_type === 'feat'`

### Data Flow

**Before:**
```
Store (race) ──► Extract modifiers ──► fixedBonuses
                                   ──► choiceBonuses
API (each feat) ──► Extract modifiers ──► featAbilityModifiers
```

**After:**
```
API (ability-bonuses) ──► Filter by source_type ──► raceBonuses (fixed only)
                                                ──► featBonuses
Store (pending choices) ──► choiceBonuses (unchanged)
```

### Key Detail

The endpoint only returns **resolved** bonuses. Unresolved choice bonuses (Half-Elf picking +1s) still come from the pending choices API - that logic stays unchanged.

### TypeScript Types

Types will be synced from backend via `npm run types:sync`. Expected shape:

```typescript
interface AbilityBonus {
  source_type: 'race' | 'feat' | 'class_feature' | 'item' | 'condition'
  source_name: string
  source_slug: string
  ability_code: 'STR' | 'DEX' | 'CON' | 'INT' | 'WIS' | 'CHA'
  ability_name: string
  value: number
  is_choice: boolean
  choice_resolved?: boolean
  modifier_id?: number
}

interface AbilityBonusesResponse {
  data: {
    bonuses: AbilityBonus[]
    totals: Record<'STR' | 'DEX' | 'CON' | 'INT' | 'WIS' | 'CHA', number>
  }
}
```

## Testing Strategy

### Test Changes

| Test Category | Change |
|---------------|--------|
| Existing feat bonus display tests | Update mocks to use new endpoint |
| Loading state tests | Remove `featsPending` tests, integrate with step loading |
| Final score calculation tests | Should pass unchanged |

### New Test Cases

1. Endpoint integration - Verify endpoint is called on mount
2. Bonus grouping - Race and feat bonuses display in correct sections
3. Empty states - Handle character with no feat bonuses
4. Error handling - Graceful fallback if endpoint fails

### Mock Shape

```typescript
mockFetch('/characters/123/ability-bonuses', {
  data: {
    bonuses: [
      { source_type: 'race', source_name: 'Elf', ability_code: 'DEX', value: 2, ... },
      { source_type: 'feat', source_name: 'Actor', ability_code: 'CHA', value: 1, ... }
    ],
    totals: { STR: 0, DEX: 2, CON: 0, INT: 0, WIS: 0, CHA: 1 }
  }
})
```

## Implementation Steps

1. Sync types - `npm run types:sync`
2. Create Nitro route - `server/api/characters/[id]/ability-bonuses.get.ts`
3. Write tests first (TDD)
4. Modify `StepAbilities.vue`
5. Run test suite - `npm run test:character`
6. Manual verification

## Files Touched

| File | Action |
|------|--------|
| `app/types/api/generated.ts` | Sync (auto-generated) |
| `server/api/characters/[id]/ability-bonuses.get.ts` | Create |
| `tests/components/character/wizard/StepAbilities.test.ts` | Modify |
| `app/components/character/wizard/StepAbilities.vue` | Modify |

---

Generated with [Claude Code](https://claude.com/claude-code)
