# Monsters Feature Design

**Date:** 2025-11-22
**Status:** ✅ Approved for Implementation
**Approach:** Minimal New Components (Approach 1)

---

## Executive Summary

Add Monsters as the 7th entity type to the D&D 5e Compendium, following existing patterns for list/detail pages. Focus on core features (CR/Type filtering, stat display) with minimal new components. Reuse existing UI patterns and test infrastructure.

**Scope:**
- Monster list page with CR/Type filters
- Monster detail page with full stat block
- 3 new components (MonsterCard, UiAccordionActions, UiAccordionTraits)
- 60-75 new tests (TDD approach)

**Not in Scope:**
- Attack damage parsing (display raw text)
- Stat block alternative view (accordion is sufficient)
- Ability score modifier calculator (show raw scores)

---

## Design Decisions

### Approach Selection

**Selected: Approach 1 - Minimal New Components**

**Why:**
1. **YAGNI Compliance** - Build only what's needed now
2. **Pattern Consistency** - Matches all 6 existing entity types
3. **Fastest to Production** - 2-3 hours vs 6-8 hours for rich combat display
4. **Test-Friendly** - Reuse existing test helpers
5. **Lower Risk** - Smaller surface area, fewer bugs

**Trade-offs Accepted:**
- ✅ Ship quickly with core features
- ⚠️ Attack data displayed as raw text (not parsed)
- ⚠️ No stat block alternative view (can add later)

**Alternatives Considered:**
- **Approach 2** (Rich Combat Display): 3x more components, complex attack parsing, higher maintenance
- **Approach 3** (Hybrid): Same as Approach 1 with explicit enhancement path

---

## Component Architecture

### 1. MonsterCard Component

**File:** `app/components/monster/MonsterCard.vue`

**Purpose:** Display monster summary on list page

**Props:**
```typescript
interface Props {
  monster: Monster // From API types
}
```

**Displays:**
- CR badge (top-left, color-coded by tier)
- Type badge (top-right, neutral gray)
- Monster name (h3, bold)
- Quick stats: Size, Alignment, AC, HP
- Legendary indicator (⭐ if has legendary_actions)
- Truncated description (150 chars, 3 lines max)
- Source footer

**Layout:**
```
┌─────────────────────────┐
│ CR 5    Type: Dragon    │
│                         │
│ Ancient Red Dragon      │
│                         │
│ 🔷 Gargantuan          │
│ ⚔️ AC 22 | ❤️ 546 HP  │
│ ⭐ Legendary           │
│                         │
│ Description preview...  │
│                         │
│ Sources: MM p.97...     │
└─────────────────────────┘
```

**Reuses:**
- `<UiCardSourceFooter>` for sources
- Card hover effects (test helper)
- Border styling (test helper)
- Link behavior (NuxtLink wrapper)

**Badge Colors:**
- CR 0-4: `success` (green - easy threat)
- CR 5-10: `info` (blue - medium threat)
- CR 11-16: `warning` (yellow - hard threat)
- CR 17+: `error` (red - deadly threat)

---

### 2. UiAccordionActions Component

**File:** `app/components/ui/accordion/UiAccordionActions.vue`

**Purpose:** Display actions/legendary actions in accordion format (reusable)

**Props:**
```typescript
interface Props {
  actions: Array<{
    name: string
    description: string
    attack_data?: string | null
    recharge?: string | null
    action_cost?: number // For legendary actions
  }>
  title: string // "Actions", "Legendary Actions", etc.
  showCost?: boolean // Show action cost for legendary actions
}
```

**Displays:**
- Accordion header: "{title} ({count})"
- Each action:
  - Name (bold, primary color)
  - Badges: Recharge (if present), Action cost (if legendary)
  - Description (full text)
  - Attack data (raw text, not parsed)

**Layout:**
```
▼ Actions (6)
  ┌─────────────────────────────────┐
  │ Bite            Recharge 5-6     │
  │ Melee Weapon Attack: +17 to hit,│
  │ reach 15 ft., one target. Hit:  │
  │ 21 (2d10+10) piercing + 14 (4d6)│
  │ fire damage.                     │
  └─────────────────────────────────┘
```

**Reuses:**
- NuxtUI `<UAccordion>` component
- NuxtUI `<UBadge>` for recharge/cost

---

### 3. UiAccordionTraits Component

**File:** `app/components/ui/accordion/UiAccordionTraits.vue`

**Purpose:** Display passive traits in accordion format (reusable)

**Props:**
```typescript
interface Props {
  traits: Array<{
    name: string
    description: string
  }>
  title?: string // Default: "Traits"
}
```

**Displays:**
- Accordion header: "{title} ({count})"
- Each trait:
  - Name (bold, primary color)
  - Description (full text)

**Layout:**
```
▼ Traits (3)
  ┌─────────────────────────────────┐
  │ Legendary Resistance (3/Day)    │
  │ If the dragon fails a saving    │
  │ throw, it can choose to succeed │
  │ instead.                         │
  └─────────────────────────────────┘
```

**Reuses:**
- Same accordion pattern as `UiAccordionBulletList`

---

## Page Architecture

### List Page: `/monsters`

**File:** `app/pages/monsters/index.vue`

**Structure:**
```vue
<template>
  <PageHeader title="Monsters" :count="pagination.total" />

  <!-- Filters -->
  <div class="filters">
    <UInput v-model="search" placeholder="Search monsters..." />
    <USelectMenu v-model="selectedCR" :options="crOptions" />
    <USelectMenu v-model="selectedType" :options="typeOptions" />
  </div>

  <ResultsCount :count="pagination.total" entity="monsters" />

  <!-- Monster Grid -->
  <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
    <MonsterCard v-for="monster in monsters" :monster="monster" />
  </div>

  <UPagination v-model:page="currentPage" :total="pagination.total" />

  <JsonDebugPanel :data="{ monsters, pagination }" />
</template>
```

**Filters:**
1. **Search** - Text input (name/description search)
2. **Challenge Rating** - Select menu:
   - All CRs
   - CR 0-4 (Easy)
   - CR 5-10 (Medium)
   - CR 11-16 (Hard)
   - CR 17+ (Deadly)
3. **Type** - Select menu (fetched from API or hardcoded):
   - All Types
   - Aberration, Beast, Celestial, Construct, Dragon, Elemental, Fey, Fiend, Giant, Humanoid, Monstrosity, Ooze, Plant, Undead

**Query Parameters:**
- `?search=dragon` - Search filter
- `?cr=5-10` - CR range filter
- `?type=dragon` - Type filter
- `?page=2` - Pagination

**Composable:**
- Uses `useEntityList<Monster>` (existing pattern)

**Pattern Matches:** `spells/index.vue`, `items/index.vue`

---

### Detail Page: `/monsters/{slug}`

**File:** `app/pages/monsters/[slug].vue`

**Structure:**
```vue
<template>
  <UiBackLink to="/monsters" label="Back to Monsters" />

  <UiDetailPageHeader
    :title="monster.name"
    :badges="[crBadge, typeBadge, legendaryBadge]"
  />

  <UiDetailQuickStatsCard :stats="quickStats" />

  <div class="description">
    {{ monster.description }}
  </div>

  <UiAccordionTraits :traits="monster.traits" />

  <UiAccordionActions
    :actions="monster.actions"
    title="Actions"
  />

  <UiAccordionActions
    v-if="monster.legendary_actions?.length"
    :actions="monster.legendary_actions"
    title="Legendary Actions"
    :showCost="true"
  />

  <UiModifiersDisplay :modifiers="monster.modifiers" />

  <UiAccordionBulletList
    v-if="monster.conditions?.length"
    :items="monster.conditions"
    title="Conditions"
  />

  <UiSourceDisplay :sources="monster.sources" />

  <JsonDebugPanel :data="monster" />
</template>
```

**Quick Stats Card Displays:**
- Size, Type, Alignment
- AC (with armor type if present)
- HP (average + hit dice)
- Speeds (walk, fly, swim, burrow, climb)
- Ability Scores (STR/DEX/CON/INT/WIS/CHA - raw values)
- Challenge Rating + XP

**Quick Stats Layout:**
```
┌─────────────────────────────────┐
│ Size: Gargantuan | Type: Dragon │
│ Alignment: Chaotic Evil         │
│ AC: 22 (natural armor)          │
│ HP: 546 (28d20+252)             │
│ Speed: 40 ft., fly 80 ft.       │
│ STR 30  DEX 10  CON 29          │
│ INT 18  WIS 15  CHA 23          │
│ CR: 24 (62,000 XP)              │
└─────────────────────────────────┘
```

**Composable:**
- Uses `useEntityDetail<Monster>` (existing pattern)

**SEO:**
- Title: "{Monster Name} - D&D 5e Monster"
- Description: First 160 chars of description

**Pattern Matches:** `spells/[slug].vue`, `items/[slug].vue`

---

## Type Definitions & Utilities

### Type Definitions

**File:** `app/types/api/entities.ts`

Add Monster type export:

```typescript
import type { components } from './generated'

// Existing exports...
export type Monster = components['schemas']['MonsterResource']
```

**Note:** MonsterResource should already exist in `generated.ts` from OpenAPI sync.

---

### Badge Color Utility

**File:** `app/utils/badgeColors.ts`

Add CR color function:

```typescript
/**
 * Get badge color for Challenge Rating
 * Maps CR to D&D difficulty tiers
 */
export function getChallengeRatingColor(cr: string): BadgeColor {
  // CR can be "1/8", "1/4", "1/2", or "1" through "30"
  const numericCR = cr.includes('/')
    ? eval(cr) as number // "1/4" → 0.25
    : parseFloat(cr)

  if (numericCR <= 4) return 'success'  // Easy
  if (numericCR <= 10) return 'info'    // Medium
  if (numericCR <= 16) return 'warning' // Hard
  return 'error'                         // Deadly
}

export type BadgeColor = 'primary' | 'secondary' | 'success' | 'info' | 'warning' | 'error' | 'neutral'
```

**Test Coverage:**
- CR 0 → success
- CR 1/8 → success
- CR 4 → success
- CR 5 → info
- CR 10 → info
- CR 11 → warning
- CR 16 → warning
- CR 17 → error
- CR 30 → error

---

## Navigation Updates

Add Monsters link to main navigation (between Items and Races):

**File:** `app/layouts/default.vue` or navigation config

```vue
<NuxtLink to="/monsters">Monsters</NuxtLink>
```

---

## Testing Strategy

### Test Files to Create

1. **`tests/components/monster/MonsterCard.test.ts`** (25-30 tests)
   - Renders CR badge with correct color
   - Renders type badge
   - Renders monster name
   - Displays size, alignment, AC, HP
   - Shows legendary indicator when has legendary_actions
   - Hides legendary indicator when no legendary_actions
   - Truncates long descriptions
   - Does not truncate short descriptions
   - Renders source footer
   - Links to detail page with slug
   - Applies hover effects
   - Uses card border styling
   - Handles missing optional fields gracefully

2. **`tests/components/ui/accordion/UiAccordionActions.test.ts`** (15-20 tests)
   - Renders accordion header with count
   - Displays action names
   - Displays action descriptions
   - Shows recharge badge when present
   - Shows action cost badge when showCost=true
   - Hides action cost when showCost=false
   - Handles empty actions array
   - Renders attack_data as raw text
   - Accordion opens/closes correctly

3. **`tests/components/ui/accordion/UiAccordionTraits.test.ts`** (10-15 tests)
   - Renders accordion header with count
   - Displays trait names
   - Displays trait descriptions
   - Handles empty traits array
   - Uses default title "Traits"
   - Uses custom title when provided
   - Accordion opens/closes correctly

4. **`tests/utils/badgeColors.test.ts`** (Add 10 tests)
   - getChallengeRatingColor: CR 0 returns success
   - getChallengeRatingColor: CR 1/8 returns success
   - getChallengeRatingColor: CR 4 returns success
   - getChallengeRatingColor: CR 5 returns info
   - getChallengeRatingColor: CR 10 returns info
   - getChallengeRatingColor: CR 11 returns warning
   - getChallengeRatingColor: CR 16 returns warning
   - getChallengeRatingColor: CR 17 returns error
   - getChallengeRatingColor: CR 30 returns error
   - getChallengeRatingColor: Handles fractional CRs

### Test Helpers Reused

From `tests/helpers/`:
- `testCardLinkBehavior()` - Link routing tests
- `testCardHoverEffects()` - Hover state tests
- `testCardBorderStyling()` - Border styling tests
- `testDescriptionTruncation()` - Description truncation tests
- `testSourceFooter()` - Source footer tests

### TDD Workflow

**Mandatory Process:**
1. **RED:** Write test first, watch it fail
2. **GREEN:** Write minimal code to pass
3. **REFACTOR:** Clean up while keeping tests green

**Estimated Total:** 60-75 new tests (maintaining 100% pass rate)

---

## API Integration

### Endpoints Used

**List Endpoint:**
- `GET /api/v1/monsters`
- Query params: `search`, `cr`, `type`, `page`, `per_page`
- Returns paginated monster list

**Detail Endpoint:**
- `GET /api/v1/monsters/{slug}`
- Returns full monster data with nested relations

**Type Endpoint (if available):**
- `GET /api/v1/monster-types` (optional)
- For populating type filter dropdown

### Monster Data Structure

```typescript
interface Monster {
  id: number
  slug: string
  name: string
  size: { id: number, code: string, name: string }
  type: string // "dragon", "humanoid (aarakocra)", etc.
  alignment: string
  armor_class: number
  armor_type: string | null
  hit_points_average: number
  hit_dice: string
  speed_walk: number
  speed_fly: number | null
  speed_swim: number | null
  speed_burrow: number | null
  speed_climb: number | null
  can_hover: boolean
  strength: number
  dexterity: number
  constitution: number
  intelligence: number
  wisdom: number
  charisma: number
  challenge_rating: string // "1/4", "5", "24", etc.
  experience_points: number
  description: string
  traits: Array<{
    id: number
    name: string
    description: string
    attack_data: string | null
    sort_order: number
  }>
  actions: Array<{
    id: number
    action_type: string
    name: string
    description: string
    attack_data: string | null
    recharge: string | null
    sort_order: number
  }>
  legendary_actions: Array<{
    id: number
    name: string
    description: string
    action_cost: number
    is_lair_action: boolean
    attack_data: string | null
    recharge: string | null
    sort_order: number
  }>
  modifiers: Array<{
    id: number
    modifier_category: string
    value: string
    condition: string | null
    is_choice: boolean
    choice_count: number | null
    choice_constraint: string | null
  }>
  conditions: Array<{
    // Condition data structure
  }>
  sources: Array<{
    id: number
    code: string
    name: string
    pages: string | null
  }>
}
```

---

## Implementation Checklist

### Phase 1: Setup & Types
- [ ] Verify Monster type exists in `app/types/api/generated.ts`
- [ ] Add Monster export to `app/types/api/entities.ts`
- [ ] Add `getChallengeRatingColor()` to `app/utils/badgeColors.ts`
- [ ] Write badge color tests (TDD)

### Phase 2: MonsterCard Component
- [ ] Write MonsterCard tests (TDD - RED phase)
- [ ] Create `app/components/monster/MonsterCard.vue`
- [ ] Implement component (GREEN phase)
- [ ] Verify all tests pass
- [ ] Refactor if needed

### Phase 3: Accordion Components
- [ ] Write UiAccordionTraits tests (TDD - RED)
- [ ] Create `app/components/ui/accordion/UiAccordionTraits.vue`
- [ ] Implement component (GREEN)
- [ ] Write UiAccordionActions tests (TDD - RED)
- [ ] Create `app/components/ui/accordion/UiAccordionActions.vue`
- [ ] Implement component (GREEN)
- [ ] Verify all tests pass

### Phase 4: List Page
- [ ] Create `app/pages/monsters/index.vue`
- [ ] Implement search functionality
- [ ] Implement CR filter
- [ ] Implement type filter
- [ ] Add pagination
- [ ] Test in browser (all filters work)
- [ ] Verify mobile responsiveness

### Phase 5: Detail Page
- [ ] Create `app/pages/monsters/[slug].vue`
- [ ] Implement quick stats card
- [ ] Add accordion sections
- [ ] Add description section
- [ ] Test in browser (all data displays)
- [ ] Verify mobile responsiveness

### Phase 6: Navigation & Polish
- [ ] Add Monsters link to navigation
- [ ] Test navigation from all pages
- [ ] Verify dark mode works
- [ ] Run full test suite (564 → 624+ tests)
- [ ] Run typecheck (maintain 13 errors or better)
- [ ] Update CHANGELOG.md
- [ ] Commit work

---

## Success Criteria

- [ ] All 60-75 new tests passing (100% pass rate maintained)
- [ ] TypeScript errors: 13 or fewer (no regression)
- [ ] ESLint: 0 errors
- [ ] List page loads and displays monsters
- [ ] Filters work (search, CR, type)
- [ ] Pagination works
- [ ] Detail page displays all monster data
- [ ] Accordion sections expand/collapse
- [ ] Navigation includes Monsters link
- [ ] Dark mode works on all new pages
- [ ] Mobile responsive (375px - 1440px+)
- [ ] CHANGELOG.md updated
- [ ] Code committed with descriptive message

---

## Future Enhancements (Not in Scope)

**v2 - Attack Parsing:**
- Parse `attack_data` JSON strings into structured damage/to-hit display
- Example: `"Bite|+17|(2d10+10)+(4d6)"` → "Bite +17 to hit, 2d10+10 piercing + 4d6 fire"

**v3 - Stat Block View:**
- Add toggle for traditional D&D stat block layout
- Side-by-side view with ability score modifiers

**v4 - Advanced Filters:**
- Size filter
- Alignment filter
- Environment/Terrain filter (if available in API)
- Legendary/Non-legendary toggle

**v5 - Ability Score Calculator:**
- Auto-calculate ability modifiers from raw scores
- Display saving throw bonuses
- Display skill bonuses with proficiency

---

## Resources

**API Documentation:**
- OpenAPI Spec: `http://localhost:8080/docs/api.json`
- API Docs: `http://localhost:8080/docs/api`

**Existing Patterns:**
- Spells: `app/pages/spells/` (similar filtering)
- Items: `app/pages/items/` (similar card layout)
- All entities: Consistent accordion/badge patterns

**Test Patterns:**
- Card tests: `tests/components/spell/SpellCard.test.ts`
- Accordion tests: `tests/components/ui/accordion/`
- Test helpers: `tests/helpers/`

**Design Documents:**
- Current Status: `docs/CURRENT_STATUS.md`
- TypeScript Cleanup: `docs/HANDOVER-2025-11-22-TYPESCRIPT-TEST-CLEANUP.md`

---

**Design Complete:** 2025-11-22
**Ready for Implementation**
