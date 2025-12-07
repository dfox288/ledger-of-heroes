# Character Stress Test Script - Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Create a Node.js script that rapidly creates N characters with random valid choices to stress-test the character builder API.

**Architecture:** Single TypeScript file in `scripts/` that calls the Laravel backend directly (port 8080). Uses curated pool of PHB races/classes/backgrounds, resolves all pending choices randomly, validates results, and reports summary.

**Tech Stack:** Node.js 18+ (native fetch), TypeScript, tsx for execution

---

## Task 1: Create Script Skeleton with CLI Parsing

**Files:**
- Create: `scripts/character-stress-test.ts`
- Modify: `package.json` (add npm script)

**Step 1: Create the script file with CLI argument parsing**

```typescript
#!/usr/bin/env npx tsx

/**
 * Character Stress Test Script
 *
 * Creates N characters with random valid choices to stress-test the character builder.
 * Run via: npm run test:character-stress -- --count=10
 */

// ════════════════════════════════════════════════════════════════
// CONFIGURATION
// ════════════════════════════════════════════════════════════════

const API_BASE = process.env.API_BASE_URL || 'http://localhost:8080/api/v1'

interface CliOptions {
  count: number
  verbose: boolean
  dryRun: boolean
}

function parseArgs(): CliOptions {
  const args = process.argv.slice(2)
  const options: CliOptions = {
    count: 10,
    verbose: false,
    dryRun: false
  }

  for (const arg of args) {
    if (arg.startsWith('--count=')) {
      options.count = parseInt(arg.split('=')[1], 10)
    } else if (arg === '--verbose' || arg === '-v') {
      options.verbose = true
    } else if (arg === '--dry-run') {
      options.dryRun = true
    } else if (arg === '--help' || arg === '-h') {
      console.log(`
Character Stress Test Script

Usage: npm run test:character-stress -- [options]

Options:
  --count=N    Number of characters to create (default: 10)
  --verbose    Show detailed output for each step
  --dry-run    Show what would happen without making API calls
  --help       Show this help message
`)
      process.exit(0)
    }
  }

  return options
}

// ════════════════════════════════════════════════════════════════
// MAIN
// ════════════════════════════════════════════════════════════════

async function main() {
  const options = parseArgs()

  console.log('🎲 Character Stress Test')
  console.log(`   Creating ${options.count} characters...`)
  console.log(`   API: ${API_BASE}`)
  if (options.dryRun) console.log('   (DRY RUN - no API calls)')
  console.log('')

  // TODO: Implement character creation loop
  console.log('⚠️  Script skeleton - implementation pending')
}

main().catch((error) => {
  console.error('❌ Fatal error:', error.message)
  process.exit(1)
})
```

**Step 2: Add npm script to package.json**

In `package.json`, add to the `"scripts"` section after the `"types:sync"` line:

```json
"test:character-stress": "npx tsx scripts/character-stress-test.ts"
```

**Step 3: Verify script runs**

Run: `docker compose exec nuxt npm run test:character-stress -- --help`

Expected output:
```
Character Stress Test Script

Usage: npm run test:character-stress -- [options]
...
```

**Step 4: Commit**

```bash
git add scripts/character-stress-test.ts package.json
git commit -m "feat: add character stress test script skeleton"
```

---

## Task 2: Add Test Pool and Random Selection Utilities

**Files:**
- Modify: `scripts/character-stress-test.ts`

**Step 1: Add the curated test pool and utility functions**

Add after the `CliOptions` interface:

```typescript
// ════════════════════════════════════════════════════════════════
// TEST POOL
// ════════════════════════════════════════════════════════════════

const TEST_POOL = {
  races: [
    'phb:human',      // Simple, 1 language choice
    'phb:elf',        // Has subraces (will use base for now)
    'phb:dwarf',      // Has subraces (will use base for now)
    'phb:half-elf',   // Multiple choices (skills, languages)
    'phb:tiefling',   // No subraces, racial spells
  ],
  classes: [
    'phb:fighter',    // Fighting style, multiple equipment choices
    'phb:wizard',     // Spellcaster with spell selection
    'phb:rogue',      // Expertise choices
    'phb:bard',       // Jack of all trades, spells
    'phb:ranger',     // Fighting style, spells
  ],
  backgrounds: [
    'phb:acolyte',    // Languages
    'phb:soldier',    // Gaming set choice
    'phb:sage',       // Languages
    'phb:criminal',   // Tool proficiency
  ]
}

// ════════════════════════════════════════════════════════════════
// UTILITIES
// ════════════════════════════════════════════════════════════════

function pickRandom<T>(arr: T[]): T {
  return arr[Math.floor(Math.random() * arr.length)]
}

function pickRandomN<T>(arr: T[], n: number): T[] {
  const shuffled = [...arr].sort(() => Math.random() - 0.5)
  return shuffled.slice(0, n)
}

function generateCharacterName(): string {
  const prefixes = ['Thorin', 'Lyria', 'Grok', 'Elara', 'Vex', 'Zara', 'Kael', 'Mira', 'Drax', 'Nova']
  const suffixes = ['Stoneshield', 'Moonwhisper', 'Ironfist', 'Shadowstep', 'Brightblade', 'Darkhollow']
  return `${pickRandom(prefixes)} ${pickRandom(suffixes)}`
}

function generatePublicId(): string {
  const adjectives = ['swift', 'brave', 'shadow', 'iron', 'storm', 'frost', 'flame', 'wild']
  const nouns = ['wolf', 'hawk', 'bear', 'dragon', 'phoenix', 'raven', 'tiger', 'serpent']
  const suffix = Math.random().toString(36).substring(2, 6)
  return `${pickRandom(adjectives)}-${pickRandom(nouns)}-${suffix}`
}
```

**Step 2: Verify script still runs**

Run: `docker compose exec nuxt npm run test:character-stress -- --count=1`

Expected: Script runs without errors, shows skeleton message.

**Step 3: Commit**

```bash
git add scripts/character-stress-test.ts
git commit -m "feat: add test pool and random utilities for stress test"
```

---

## Task 3: Implement API Helper Functions

**Files:**
- Modify: `scripts/character-stress-test.ts`

**Step 1: Add API helper functions**

Add after the utilities section:

```typescript
// ════════════════════════════════════════════════════════════════
// API HELPERS
// ════════════════════════════════════════════════════════════════

interface ApiError extends Error {
  status?: number
  body?: unknown
}

async function apiFetch<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const url = `${API_BASE}${endpoint}`

  const response = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
      ...options.headers
    }
  })

  if (!response.ok) {
    const error: ApiError = new Error(`API error: ${response.status} ${response.statusText}`)
    error.status = response.status
    try {
      error.body = await response.json()
    } catch {
      // Response wasn't JSON
    }
    throw error
  }

  return response.json()
}

async function createCharacter(name: string, publicId: string, raceSlug: string): Promise<{ id: number; public_id: string }> {
  const response = await apiFetch<{ data: { id: number; public_id: string } }>('/characters', {
    method: 'POST',
    body: JSON.stringify({
      name,
      public_id: publicId,
      race_slug: raceSlug
    })
  })
  return response.data
}

async function addClass(characterId: number, classSlug: string): Promise<void> {
  await apiFetch(`/characters/${characterId}/classes`, {
    method: 'POST',
    body: JSON.stringify({
      class_slug: classSlug,
      force: true  // Skip multiclass prereqs for initial class
    })
  })
}

async function updateCharacter(characterId: number, data: Record<string, unknown>): Promise<void> {
  await apiFetch(`/characters/${characterId}`, {
    method: 'PATCH',
    body: JSON.stringify(data)
  })
}

async function deleteCharacter(characterId: number): Promise<void> {
  await apiFetch(`/characters/${characterId}`, {
    method: 'DELETE'
  })
}

async function getValidation(characterId: number): Promise<{ is_complete: boolean; missing: string[] }> {
  const response = await apiFetch<{ data: { validation_status: { is_complete: boolean; missing: string[] } } }>(
    `/characters/${characterId}`
  )
  return response.data.validation_status
}

async function getStats(characterId: number): Promise<Record<string, unknown>> {
  const response = await apiFetch<{ data: Record<string, unknown> }>(`/characters/${characterId}/stats`)
  return response.data
}

async function getSummary(characterId: number): Promise<{ creation_complete: boolean; missing_required: string[] }> {
  const response = await apiFetch<{ data: { creation_complete: boolean; missing_required: string[] } }>(
    `/characters/${characterId}/summary`
  )
  return response.data
}
```

**Step 2: Test API connectivity**

Run: `docker compose exec nuxt npm run test:character-stress -- --count=1 --verbose`

(Still shows skeleton message, but API helpers are ready)

**Step 3: Commit**

```bash
git add scripts/character-stress-test.ts
git commit -m "feat: add API helper functions for character stress test"
```

---

## Task 4: Implement Pending Choices Fetching and Resolution

**Files:**
- Modify: `scripts/character-stress-test.ts`

**Step 1: Add types and choice-related functions**

Add after the API helpers:

```typescript
// ════════════════════════════════════════════════════════════════
// CHOICE TYPES
// ════════════════════════════════════════════════════════════════

interface ChoiceOption {
  full_slug?: string
  slug?: string
  name: string
  option?: string  // "a" or "b" for equipment
  items?: Array<{ full_slug: string; is_fixed: boolean }>
  is_category?: boolean
  is_fixed?: boolean
}

interface PendingChoice {
  id: string
  type: string
  subtype?: string
  source: string
  source_name: string
  required: boolean
  quantity: number
  remaining: number
  selected: string[]
  options: ChoiceOption[]
  options_endpoint?: string
  metadata?: {
    known_languages?: string[]
    choice_group?: string
  }
}

interface PendingChoicesResponse {
  data: {
    choices: PendingChoice[]
    summary: {
      total_pending: number
      required_pending: number
    }
  }
}

// ════════════════════════════════════════════════════════════════
// CHOICE RESOLUTION
// ════════════════════════════════════════════════════════════════

async function getPendingChoices(characterId: number): Promise<PendingChoice[]> {
  const response = await apiFetch<PendingChoicesResponse>(`/characters/${characterId}/pending-choices`)
  return response.data.choices
}

async function resolveChoice(
  characterId: number,
  choiceId: string,
  payload: { selected: string | string[]; items?: string[] }
): Promise<void> {
  const encodedId = encodeURIComponent(choiceId)
  await apiFetch(`/characters/${characterId}/choices/${encodedId}`, {
    method: 'POST',
    body: JSON.stringify(payload)
  })
}

function pickRandomChoiceSelection(choice: PendingChoice): { selected: string | string[]; items?: string[] } {
  switch (choice.type) {
    case 'proficiency':
    case 'language': {
      // Filter out already-known items if metadata provides them
      const known = choice.metadata?.known_languages || []
      const available = choice.options
        .map(o => o.full_slug || o.slug)
        .filter((slug): slug is string => !!slug && !known.includes(slug))

      if (available.length < choice.remaining) {
        console.warn(`    ⚠️  Not enough options for ${choice.type}: need ${choice.remaining}, have ${available.length}`)
        return { selected: available }
      }

      return { selected: pickRandomN(available, choice.remaining) }
    }

    case 'equipment': {
      // Equipment choices have options like {option: "a", items: [...]}
      const equipOption = pickRandom(choice.options)

      if (equipOption.is_category && equipOption.items) {
        // Need to pick specific items from category
        const selectableItems = equipOption.items.filter(i => !i.is_fixed)
        const fixedCount = equipOption.items.filter(i => i.is_fixed).length

        // Pick enough items to satisfy quantity minus fixed items
        const needed = Math.max(1, choice.quantity - fixedCount)
        const pickedItems = pickRandomN(
          selectableItems.map(i => i.full_slug),
          Math.min(needed, selectableItems.length)
        )

        return {
          selected: equipOption.option!,
          items: pickedItems
        }
      }

      // Simple a/b choice
      return { selected: equipOption.option! }
    }

    case 'spell':
    case 'fighting_style':
    case 'optional_feature': {
      const available = choice.options
        .map(o => o.full_slug || o.slug)
        .filter((slug): slug is string => !!slug)

      if (available.length === 0 && choice.options_endpoint) {
        // Would need to fetch from endpoint - for now, skip
        console.warn(`    ⚠️  Choice ${choice.id} requires endpoint fetch (not implemented)`)
        return { selected: [] }
      }

      return { selected: pickRandomN(available, choice.remaining) }
    }

    default:
      console.warn(`    ⚠️  Unknown choice type: ${choice.type}`)
      return { selected: [] }
  }
}

async function resolveAllChoices(characterId: number, verbose: boolean): Promise<number> {
  let resolved = 0
  let iterations = 0
  const maxIterations = 20  // Safety limit

  while (iterations < maxIterations) {
    iterations++
    const choices = await getPendingChoices(characterId)
    const pendingRequired = choices.filter(c => c.required && c.remaining > 0)

    if (pendingRequired.length === 0) {
      if (verbose) console.log(`    ✓ All choices resolved after ${iterations} iteration(s)`)
      break
    }

    if (verbose) {
      console.log(`    📋 Iteration ${iterations}: ${pendingRequired.length} choices remaining`)
    }

    for (const choice of pendingRequired) {
      try {
        const selection = pickRandomChoiceSelection(choice)

        if (verbose) {
          const selStr = Array.isArray(selection.selected)
            ? selection.selected.join(', ')
            : selection.selected
          console.log(`       → ${choice.type} (${choice.source_name}): ${selStr}`)
        }

        // Skip empty selections
        if (
          (Array.isArray(selection.selected) && selection.selected.length === 0) ||
          selection.selected === ''
        ) {
          continue
        }

        await resolveChoice(characterId, choice.id, selection)
        resolved++
      } catch (err) {
        const msg = err instanceof Error ? err.message : String(err)
        console.warn(`    ⚠️  Failed to resolve ${choice.id}: ${msg}`)
      }
    }
  }

  if (iterations >= maxIterations) {
    console.warn(`    ⚠️  Hit max iterations (${maxIterations}) - some choices may be unresolved`)
  }

  return resolved
}
```

**Step 2: Verify compilation**

Run: `docker compose exec nuxt npx tsx --version` (ensures tsx is available)

**Step 3: Commit**

```bash
git add scripts/character-stress-test.ts
git commit -m "feat: add pending choices fetching and resolution"
```

---

## Task 5: Implement Main Loop and Reporting

**Files:**
- Modify: `scripts/character-stress-test.ts`

**Step 1: Add result types and update main function**

Add after the choice resolution section:

```typescript
// ════════════════════════════════════════════════════════════════
// RESULT TRACKING
// ════════════════════════════════════════════════════════════════

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
    missing: string[]
  }
}

function formatDuration(ms: number): string {
  return `${(ms / 1000).toFixed(2)}s`
}

function printSummary(results: CharacterResult[]): void {
  const successful = results.filter(r => r.status === 'success')
  const failed = results.filter(r => r.status === 'error')
  const avgTime = results.reduce((sum, r) => sum + r.timing.total, 0) / results.length

  console.log('')
  console.log('═'.repeat(65))
  console.log(`Summary: ${successful.length}/${results.length} successful (${Math.round(successful.length / results.length * 100)}%)`)
  console.log(`Average time: ${formatDuration(avgTime)} per character`)

  if (failed.length > 0) {
    console.log('')
    console.log('Errors:')
    for (const result of failed) {
      console.log(`  - [${result.iteration}] ${result.race}/${result.class}/${result.background}: ${result.error}`)
    }
  }

  const incomplete = successful.filter(r => !r.validation.isComplete)
  if (incomplete.length > 0) {
    console.log('')
    console.log('Incomplete (missing required):')
    for (const result of incomplete) {
      console.log(`  - [${result.iteration}] ${result.name}: ${result.validation.missing.join(', ')}`)
    }
  }

  console.log('═'.repeat(65))
}
```

**Step 2: Replace the main function**

Replace the existing `main()` function with:

```typescript
async function main() {
  const options = parseArgs()

  console.log('🎲 Character Stress Test')
  console.log(`   Creating ${options.count} characters...`)
  console.log(`   API: ${API_BASE}`)
  if (options.dryRun) console.log('   (DRY RUN - no API calls)')
  console.log('')

  if (options.dryRun) {
    console.log('Dry run - showing what would be created:')
    for (let i = 1; i <= options.count; i++) {
      const race = pickRandom(TEST_POOL.races)
      const cls = pickRandom(TEST_POOL.classes)
      const bg = pickRandom(TEST_POOL.backgrounds)
      const name = generateCharacterName()
      console.log(`  [${i}] "${name}" (${race.split(':')[1]}/${cls.split(':')[1]}/${bg.split(':')[1]})`)
    }
    return
  }

  const results: CharacterResult[] = []

  for (let i = 1; i <= options.count; i++) {
    const race = pickRandom(TEST_POOL.races)
    const cls = pickRandom(TEST_POOL.classes)
    const background = pickRandom(TEST_POOL.backgrounds)
    const name = generateCharacterName()
    const publicId = generatePublicId()

    const result: CharacterResult = {
      iteration: i,
      name,
      race: race.split(':')[1],
      class: cls.split(':')[1],
      background: background.split(':')[1],
      status: 'success',
      timing: { create: 0, choices: 0, validate: 0, total: 0 },
      validation: { isComplete: false, missing: [] }
    }

    const startTotal = Date.now()
    let characterId: number | null = null

    try {
      // Step 1: Create character with race
      const startCreate = Date.now()
      const created = await createCharacter(name, publicId, race)
      characterId = created.id
      result.timing.create = Date.now() - startCreate

      if (options.verbose) {
        console.log(`[${i}/${options.count}] Creating "${name}" (${result.race}/${result.class}/${result.background})...`)
      }

      // Step 2: Add class
      await addClass(characterId, cls)

      // Step 3: Set background and ability scores
      await updateCharacter(characterId, {
        background_slug: background,
        ability_score_method: 'standard_array',
        strength: 15,
        dexterity: 14,
        constitution: 13,
        intelligence: 12,
        wisdom: 10,
        charisma: 8
      })

      // Step 4: Resolve all pending choices
      const startChoices = Date.now()
      await resolveAllChoices(characterId, options.verbose)
      result.timing.choices = Date.now() - startChoices

      // Step 5: Validate
      const startValidate = Date.now()
      const summary = await getSummary(characterId)
      result.validation.isComplete = summary.creation_complete
      result.validation.missing = summary.missing_required || []

      // Also get stats to verify calculations work
      await getStats(characterId)
      result.timing.validate = Date.now() - startValidate

    } catch (err) {
      result.status = 'error'
      result.error = err instanceof Error ? err.message : String(err)
    } finally {
      result.timing.total = Date.now() - startTotal

      // Cleanup: delete the character
      if (characterId) {
        try {
          await deleteCharacter(characterId)
        } catch {
          // Ignore cleanup errors
        }
      }
    }

    results.push(result)

    // Print result line
    const statusIcon = result.status === 'success'
      ? (result.validation.isComplete ? '✓' : '⚠')
      : '✗'
    const statusColor = result.status === 'success' ? '' : ' - ERROR'
    console.log(
      `[${i}/${options.count}] ${statusIcon} "${name}" (${result.race}/${result.class}/${result.background}) - ${formatDuration(result.timing.total)}${statusColor}`
    )

    if (result.status === 'error' && options.verbose) {
      console.log(`         ${result.error}`)
    }
  }

  printSummary(results)
}
```

**Step 3: Run the script with 1 character**

Run: `docker compose exec nuxt npm run test:character-stress -- --count=1 --verbose`

Expected: Creates a character, resolves choices, validates, deletes, shows summary.

**Step 4: Run with 5 characters**

Run: `docker compose exec nuxt npm run test:character-stress -- --count=5`

Expected: Creates 5 characters, shows progress, summary at end.

**Step 5: Commit**

```bash
git add scripts/character-stress-test.ts
git commit -m "feat: implement main loop and reporting for character stress test"
```

---

## Task 6: Handle Edge Cases and Improve Error Messages

**Files:**
- Modify: `scripts/character-stress-test.ts`

**Step 1: Add API connectivity check at startup**

Add this function after the API helpers and call it at the start of `main()`:

```typescript
async function checkApiConnectivity(): Promise<boolean> {
  try {
    await apiFetch('/races?per_page=1')
    return true
  } catch (err) {
    console.error('❌ Cannot connect to API at', API_BASE)
    console.error('')
    console.error('💡 Make sure the backend is running:')
    console.error('   cd ../importer && docker compose up -d')
    console.error('')
    return false
  }
}
```

Update the start of `main()` after the header output:

```typescript
  // Check API connectivity
  if (!options.dryRun) {
    const connected = await checkApiConnectivity()
    if (!connected) {
      process.exit(1)
    }
    console.log('✓ API connected')
    console.log('')
  }
```

**Step 2: Add better error context for API errors**

Update the `apiFetch` function to include more context:

```typescript
async function apiFetch<T>(
  endpoint: string,
  options: RequestInit = {}
): Promise<T> {
  const url = `${API_BASE}${endpoint}`

  const response = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      'Accept': 'application/json',
      ...options.headers
    }
  })

  if (!response.ok) {
    let errorBody: unknown
    let errorMessage = `API error: ${response.status} ${response.statusText}`

    try {
      errorBody = await response.json()
      // Extract Laravel validation errors
      if (errorBody && typeof errorBody === 'object' && 'message' in errorBody) {
        errorMessage = (errorBody as { message: string }).message
      }
      if (errorBody && typeof errorBody === 'object' && 'errors' in errorBody) {
        const errors = (errorBody as { errors: Record<string, string[]> }).errors
        const firstError = Object.values(errors)[0]?.[0]
        if (firstError) errorMessage = firstError
      }
    } catch {
      // Response wasn't JSON
    }

    const error: ApiError = new Error(errorMessage)
    error.status = response.status
    error.body = errorBody
    throw error
  }

  return response.json()
}
```

**Step 3: Test with API down (optional manual test)**

Stop backend, run script, verify helpful error message appears.

**Step 4: Commit**

```bash
git add scripts/character-stress-test.ts
git commit -m "feat: add API connectivity check and improved error messages"
```

---

## Task 7: Final Testing and Documentation

**Files:**
- Modify: `scripts/character-stress-test.ts` (add header comment)

**Step 1: Add comprehensive header comment**

Update the header comment at the top of the file:

```typescript
#!/usr/bin/env npx tsx

/**
 * Character Stress Test Script
 *
 * Creates N characters with random valid choices to stress-test the character builder API.
 * Uses a curated pool of PHB races/classes/backgrounds for reliable testing.
 *
 * Usage:
 *   npm run test:character-stress -- --count=10
 *   npm run test:character-stress -- --count=5 --verbose
 *   npm run test:character-stress -- --count=1 --dry-run
 *
 * Options:
 *   --count=N    Number of characters to create (default: 10)
 *   --verbose    Show detailed output for each step
 *   --dry-run    Show what would happen without making API calls
 *   --help       Show help message
 *
 * Environment:
 *   API_BASE_URL  Override API base URL (default: http://localhost:8080/api/v1)
 *
 * Test Pool:
 *   Races: human, elf, dwarf, half-elf, tiefling (5)
 *   Classes: fighter, wizard, rogue, bard, ranger (5)
 *   Backgrounds: acolyte, soldier, sage, criminal (4)
 *   = 100 unique combinations
 *
 * What it tests:
 *   - Character creation with race
 *   - Adding a class
 *   - Setting background and ability scores
 *   - Resolving all pending choices (proficiencies, languages, equipment, spells)
 *   - Validation endpoint
 *   - Stats calculation endpoint
 *   - Summary endpoint
 *
 * @see docs/plans/2025-12-07-character-stress-test-script.md
 */
```

**Step 2: Run full test with 10 characters**

Run: `docker compose exec nuxt npm run test:character-stress -- --count=10`

Expected: All 10 characters created and validated successfully (or clear error messages for any failures).

**Step 3: Final commit**

```bash
git add scripts/character-stress-test.ts
git commit -m "docs: add comprehensive header documentation for stress test script"
```

---

## Summary

After completing all tasks, you will have:

1. **`scripts/character-stress-test.ts`** - Complete stress test script
2. **`package.json`** - New `test:character-stress` npm script
3. **Design doc** - Already created at `docs/frontend/plans/2025-12-07-character-stress-test-script.md`

**Usage:**
```bash
# Quick test
npm run test:character-stress -- --count=3

# Full stress test
npm run test:character-stress -- --count=10

# Debug mode
npm run test:character-stress -- --count=1 --verbose

# Preview without API calls
npm run test:character-stress -- --count=5 --dry-run
```

**Future enhancements** (documented in design doc):
- Level 1 subclass selection (Cleric, Sorcerer, Warlock)
- Subrace selection
- Comprehensive mode (fetch all entities from API)
- Seed support for reproducible runs
- Parallel execution
