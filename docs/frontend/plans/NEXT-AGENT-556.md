# Next Agent: Implement Spells Tab (#556)

## Quick Start

```bash
cd /Users/dfox/Development/ledger-of-heroes/frontend
git checkout feature/issue-556-spells-tab
```

## Your Task

Implement the Spells Tab following the detailed plan at:
**`wrapper/docs/frontend/plans/2025-12-14-spells-tab-implementation.md`**

## Required Skill

Use **`superpowers:executing-plans`** to work through the implementation plan task-by-task.

## TDD is Mandatory

1. Write failing tests FIRST
2. Run tests to confirm they fail
3. Write minimal code to pass
4. Run tests to confirm they pass
5. Commit

## Tasks Overview

| # | Task | Files |
|---|------|-------|
| 1 | Crystal icons for SpellSlots | `SpellSlots.vue` |
| 2 | SpellCard component | `SpellCard.vue` + tests |
| 3 | Spells page | `spells.vue` + tests + Nitro route |
| 4 | Navigation link | `index.vue` |
| 5 | PR | Full test suite |

## Test Commands

```bash
# Run specific test file
docker compose exec nuxt npm run test -- tests/components/character/sheet/SpellCard.test.ts

# Run character suite (includes spells)
docker compose exec nuxt npm run test:character

# Typecheck
docker compose exec nuxt npm run typecheck
```

## API Endpoints Available

- `GET /characters/{id}/spells` - Character's spells
- `GET /characters/{id}/spell-slots` - Slot tracking (total/spent/available)
- `GET /characters/{id}/stats` - Spellcasting info (DC, attack, ability)

## Design Decisions Made

- **Icon:** `i-game-icons-crystal-shine` at `w-7 h-7`
- **Page route:** `/characters/[publicId]/spells`
- **Cantrips:** Separate section at top
- **Spell cards:** Expandable with click
- **Badges:** Concentration, Ritual, Always (for always-prepared)

## When Done

1. All tests pass
2. Typecheck passes
3. Manual browser test (light/dark mode)
4. Create PR with `Closes #556`

## Related Issues

- #556 - This issue (Spells Tab)
- #612 - Backend work for Phase 2 (interactive features)
