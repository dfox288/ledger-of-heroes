# Character Stress Test Script

**Date:** 2025-12-07
**Status:** Design Complete
**Purpose:** Automate character creation to rapidly test the character builder flow without UI interaction

## Problem Statement

Testing the character builder through the UI is tedious and time-consuming. Each character requires clicking through multiple wizard steps, making choices, and verifying results. We need a script that can create N characters with random valid choices to stress-test the system.

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Technology | Node.js TypeScript in `scripts/` | Reuses existing types, familiar tooling |
| API Target | Direct to Laravel (port 8080) | Faster, tests backend in isolation |
| Validation Level | Deep (stats + summary + validate) | Catches calculation bugs, not just API errors |
| Randomization | Curated pool + random selection | Faster, more controlled than fetching all entities |
| Error Handling | Continue + report | Get full picture of failures across iterations |

## Script Interface

```bash
# Basic usage
npm run test:character-stress -- --count=10

# With options
npm run test:character-stress -- --count=5 --verbose
npm run test:character-stress -- --count=1 --dry-run
```

## Architecture

### Flow

```
1. Load curated pool (races, classes, backgrounds)
2. For each iteration (1..count):
   a. Pick random race/class/background from pool
   b. Create character via POST /characters
   c. Add class via POST /characters/{id}/classes
   d. Set background + ability scores via PATCH /characters/{id}
   e. Fetch pending-choices via GET /characters/{id}/pending-choices
   f. Resolve each choice with random valid selection
   g. Validate via GET /validate, /stats, /summary
   h. Log result (success/failure + timing)
   i. Delete character (cleanup)
3. Print summary report
```

### Curated Test Pool

Focused on PHB content for reliability:

```typescript
const TEST_POOL = {
  races: [
    'phb:human',      // Simple, 1 language choice
    'phb:elf',        // Has subraces (high elf, wood elf, drow)
    'phb:dwarf',      // Has subraces, different proficiencies
    'phb:half-elf',   // Multiple choices (skills, languages)
    'phb:tiefling',   // No subraces, racial spells
  ],
  classes: [
    'phb:fighter',    // Fighting style, multiple equipment choices
    'phb:wizard',     // Spellcaster with spell selection
    'phb:cleric',     // Level 1 subclass + spells (FUTURE)
    'phb:rogue',      // Expertise choices
    'phb:bard',       // Jack of all trades, spells
  ],
  backgrounds: [
    'phb:acolyte',    // Languages
    'phb:soldier',    // Gaming set choice
    'phb:sage',       // Languages
    'phb:criminal',   // Tool proficiency
  ]
}
```

This gives 5×5×4 = **100 unique combinations**.

### Choice Resolution Strategy

Each choice type has specific resolution logic:

#### Proficiency (skills, tools)
```typescript
// Pick N random from available options
const skillSlugs = choice.options.map(o => o.full_slug)
return { selected: pickRandom(skillSlugs, choice.remaining) }
```

#### Language
```typescript
// Exclude already-known languages
const knownLanguages = choice.metadata?.known_languages || []
const available = choice.options
  .filter(o => !knownLanguages.includes(o.full_slug))
return { selected: pickRandom(available, choice.remaining) }
```

#### Equipment
```typescript
// Pick option "a" or "b"
const option = randomChoice(choice.options)

if (option.is_category) {
  // Category: pick specific items (e.g., "a martial weapon")
  const selectable = option.items.filter(i => !i.is_fixed)
  return { selected: option.option, items: pickRandom(selectable, needed) }
}

return { selected: option.option }
```

#### Spell
```typescript
// Use options_endpoint if provided, else inline options
if (choice.options_endpoint) {
  const spells = await fetch(choice.options_endpoint)
  return { selected: pickRandom(spells, choice.remaining) }
}
return { selected: pickRandom(choice.options, choice.remaining) }
```

#### Fighting Style / Optional Feature
```typescript
// Pick one from available
return { selected: pickRandom(choice.options, 1) }
```

### Output Format

```
[1/10] ✓ "Thorin Stoneshield" (dwarf/fighter/soldier) - 1.2s
[2/10] ✓ "Lyria Moonwhisper" (elf/wizard/sage) - 1.5s
[3/10] ✗ "Test Character" (half-elf/cleric/acolyte) - ERROR: skill already granted
...
═══════════════════════════════════════════════════════════════════
Summary: 9/10 successful (90%)
Average time: 1.3s per character
Errors:
  - [3] half-elf/cleric/acolyte: "Invalid skill selection: arcana already granted"
═══════════════════════════════════════════════════════════════════
```

### Result Data Structure

```typescript
interface CharacterResult {
  iteration: number
  name: string
  race: string
  class: string
  background: string
  status: 'success' | 'error'
  error?: string
  timing: {
    create: number
    choices: number
    validate: number
    total: number
  }
  validation: {
    isComplete: boolean
    missingRequired: string[]
    statsValid: boolean
  }
}
```

## Future Enhancements

1. **Level 1 Subclass Selection** - Cleric, Sorcerer, Warlock pick subclass at creation
2. **Subrace Selection** - Currently uses base race; could pick random subrace
3. **Comprehensive Mode** - Fetch all races/classes from API instead of curated pool
4. **Seed Support** - Reproducible random runs with `--seed=12345`
5. **Parallel Execution** - Create multiple characters concurrently

## File Location

`scripts/character-stress-test.ts`

## Dependencies

- Node.js with TypeScript
- Native fetch (Node 18+)
- Existing types from `app/types/api/generated.ts`
