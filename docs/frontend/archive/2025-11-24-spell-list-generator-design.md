# Spell List Generator - Design Document

**Date:** 2025-11-24
**Status:** Design Approved
**Approach:** Single-Page Wizard (Approach 1)

---

## Overview

A spell list generator tool that helps players choose the right spells for their D&D character based on class, level, and spellcasting type (prepared vs known). The MVP focuses on core functionality: character setup, spell selection with limits, and LocalStorage persistence.

---

## Goals

### Primary Goals
1. **Help players make informed spell choices** - Show exactly what's available at their level
2. **Enforce D&D rules** - Respect spell slot limits, prepared/known spell counts
3. **Save progress** - Persist selections across sessions via LocalStorage
4. **Support all spellcasting classes** - Handle prepared casters, known casters, half-casters, third-casters

### Non-Goals (Out of Scope for MVP)
- ❌ Print/PDF export
- ❌ Share links (URL encoding)
- ❌ Spell comparison side-by-side
- ❌ Subclass spell lists (bonus spells)
- ❌ Ability score input (will default to +3 modifier)
- ❌ Spell slot tracking during play

---

## User Flow

```
1. Navigate to /spells/list-generator
2. Select Class (e.g., Wizard) → Auto-load spellcasting info
3. Select Level (1-20) → Calculate spell slots & max prepared/known
4. Browse available spells grouped by level (Cantrips, 1st, 2nd, etc.)
5. Check/uncheck spells → Track count vs. limits
6. Selections auto-save to LocalStorage
7. Return later → Selections persist
```

---

## Page Structure

### File Location
`/app/pages/spells/list-generator.vue`

### Layout Sections

```
┌─────────────────────────────────────────────────┐
│  1. Character Setup (always visible)            │
│     - Class dropdown                             │
│     - Level dropdown                             │
│     - Spell slots display                        │
│     - Prepared/Known count                       │
├─────────────────────────────────────────────────┤
│  2. Spell Selection (scrollable)                │
│     - Accordion per spell level                  │
│     - Checkboxes for each spell                  │
│     - Count tracker per level                    │
├─────────────────────────────────────────────────┤
│  3. Summary Sidebar (sticky, right side)        │
│     - Selected spells list                       │
│     - Save/Clear actions                         │
└─────────────────────────────────────────────────┘
```

---

## Data Architecture

### Composable: `useSpellListGenerator`

**Location:** `/app/composables/useSpellListGenerator.ts`

**Responsibilities:**
- Track selected class and character level
- Calculate spell slots from `level_progression` array
- Calculate max prepared/known spells
- Manage spell selections (Set of spell IDs)
- LocalStorage save/load
- Selection limit validation

**State:**
```typescript
{
  selectedClass: Ref<CharacterClass | null>
  characterLevel: Ref<number>
  selectedSpells: Ref<Set<number>>  // spell IDs
}
```

**Computed:**
```typescript
{
  spellSlots: { cantrips: 4, 1st: 4, 2nd: 3, 3rd: 2, ... }
  maxPrepared: number  // e.g., 9 for Wizard level 5
  availableSpellLevels: number[]  // [0, 1, 2, 3] for level 5
  selectionsByLevel: Map<number, number>  // { 0: 3, 1: 2 }
}
```

**Methods:**
```typescript
toggleSpell(spellId: number, spellLevel: number): void
canSelectMore(spellLevel: number): boolean
saveToStorage(): void
loadFromStorage(): void
clearAll(): void
```

---

## Class Type Logic

### Prepared Casters
**Classes:** Wizard, Cleric, Druid, Paladin, Artificer
**Formula:** `level + ability_modifier` (default +3 for MVP)
**Example:** Wizard level 5 = 5 + 3 = 8 spells prepared

### Known Casters
**Classes:** Bard, Sorcerer, Warlock, Ranger, Arcane Trickster, Eldritch Knight
**Formula:** Fixed number from class progression table (hardcoded)
**Example:** Bard level 5 = 8 spells known (from table)

### Known Spells Tables

```typescript
const KNOWN_SPELLS_BY_CLASS = {
  'bard': [4, 5, 6, 7, 8, 9, 10, 11, 12, 14, 15, 15, 16, 18, 19, 19, 20, 22, 22, 22],
  'sorcerer': [2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 12, 13, 13, 14, 14, 15, 15, 15, 15],
  'warlock': [2, 3, 4, 5, 6, 7, 8, 9, 10, 10, 11, 11, 12, 12, 13, 13, 14, 14, 15, 15],
  'ranger': [0, 2, 3, 3, 4, 4, 5, 5, 6, 6, 7, 7, 8, 8, 9, 9, 10, 10, 11, 11],
  'eldritch-knight': [0, 0, 2, 3, 3, 4, 4, 4, 4, 5, 6, 6, 7, 7, 7, 8, 8, 8, 9, 9],
  'arcane-trickster': [0, 0, 2, 3, 3, 4, 4, 4, 4, 5, 6, 6, 7, 7, 7, 8, 8, 8, 9, 9]
}
```

---

## Data Sources

### Classes API
**Endpoint:** `GET /classes?is_spellcaster=1`
**Returns:** List of spellcasting classes
**Key Fields:**
- `slug` (e.g., "wizard")
- `name` (e.g., "Wizard")
- `level_progression` (array with spell slots per level)
- `spellcasting_ability` (INT/WIS/CHA)

### Spells API
**Endpoint:** `GET /spells?classes={class-slug}`
**Returns:** Spells available to that class
**Key Fields:**
- `id`, `name`, `level`, `school`
- `range`, `components`, `concentration`, `ritual`

### Level Progression Structure
```json
{
  "level": 5,
  "cantrips_known": 4,
  "spells_known": null,
  "spell_slots_1st": 4,
  "spell_slots_2nd": 3,
  "spell_slots_3rd": 2,
  "spell_slots_4th": 0,
  ...
}
```

---

## UI Components

### Reuse Existing Components
- `<USelectMenu>` - Class and level dropdowns
- `<UAccordion>` - Group spells by level
- `<UCheckbox>` - Spell selection
- `<UBadge>` - Spell schools, ritual, concentration
- `<SpellCard>` - Existing spell card component (or simplified version)
- `<UButton>` - Save, Clear actions
- `<UiListErrorState>` - Error handling
- `<UiListSkeletonCards>` - Loading state

### New Components (if needed)
- `<SpellListSummary>` - Sticky sidebar with selected spells
- `<SpellSelectionCard>` - Lightweight card with checkbox + key spell info

---

## LocalStorage Schema

**Key Pattern:** `spell-list-{classSlug}-{level}`
**Example Key:** `spell-list-wizard-5`

**Value:**
```json
{
  "classSlug": "wizard",
  "className": "Wizard",
  "level": 5,
  "selectedSpells": [1, 5, 12, 45, 67, 89, 123],
  "savedAt": "2025-11-24T19:30:00Z"
}
```

**Behavior:**
- Auto-save on every selection change (debounced 500ms)
- Auto-load on page mount if key exists
- Clear all removes from localStorage
- Multiple lists supported (one per class/level combo)

---

## Selection Rules

### Per-Level Limits

**Cantrips:**
- No limit on selections (can select all available)
- But show count: "Selected 4 cantrips (you know 4)"

**Leveled Spells (1st-9th):**
- **Prepared Casters**: No per-level limit, just total limit
  - Show: "Selected 7 spells (prepare 9 total)"
- **Known Casters**: No per-level limit, just total limit
  - Show: "Selected 6 spells (know 8 total)"

**Spell Slots Display:**
- Show spell slot counts per level (informational only, not enforced)
- e.g., "1st: 4 slots, 2nd: 3 slots, 3rd: 2 slots"

---

## Error Handling

### Cases to Handle

1. **No classes available** - Show error, link to classes page
2. **API failure** - Use `<UiListErrorState>` with retry button
3. **No spells for class** - Show empty state with message
4. **LocalStorage quota exceeded** - Catch exception, show toast notification
5. **Invalid saved data** - Clear and start fresh

### Loading States
- Show skeleton cards while fetching classes
- Show skeleton cards while fetching spells
- Disable checkboxes while saving

---

## Mobile Responsiveness

### Desktop (≥1024px)
```
┌────────────────┬─────────────┐
│                │             │
│  Spell List    │   Summary   │
│  (scrollable)  │  (sticky)   │
│                │             │
└────────────────┴─────────────┘
```

### Tablet (768-1023px)
```
┌────────────────┬──────┐
│                │      │
│  Spell List    │ Sum  │
│  (scrollable)  │ mary │
│                │      │
└────────────────┴──────┘
```

### Mobile (<768px)
```
┌─────────────────────┐
│   Character Setup   │
├─────────────────────┤
│                     │
│    Spell List       │
│    (scrollable)     │
│                     │
├─────────────────────┤
│  Summary (sticky    │
│  bottom sheet)      │
└─────────────────────┘
```

---

## Testing Strategy

### Unit Tests (Composable)
**File:** `tests/composables/useSpellListGenerator.test.ts`

- ✅ Calculate spell slots from level_progression
- ✅ Calculate max prepared for prepared casters
- ✅ Calculate max known for known casters
- ✅ Toggle spell selection
- ✅ Validate selection limits
- ✅ LocalStorage save/load
- ✅ Clear all selections

### Component Tests
**File:** `tests/pages/spells/list-generator.test.ts`

- ✅ Mount with class selection
- ✅ Fetch and display spells
- ✅ Group spells by level
- ✅ Checkbox interactions
- ✅ Selection count updates
- ✅ Save/load from localStorage
- ✅ Error states

### E2E Tests (Optional)
- Full user flow: select class → select level → pick spells → save → reload page → verify persistence

---

## Future Enhancements (Post-MVP)

### Phase 2 Features
- 🎯 Spell comparison (side-by-side 2-3 spells)
- 🎯 Print/PDF export with spell cards
- 🎯 Share link (URL-encoded selections)
- 🎯 Ability score input (custom modifier)
- 🎯 Subclass bonus spells

### Phase 3 Features
- 🎯 Spell filtering (school, range, damage type)
- 🎯 Recommended builds ("Evocation Specialist", "Battlefield Control")
- 🎯 Multi-character support (save multiple lists)
- 🎯 Spell slot tracker (mark used slots during play)

### Phase 4 Features
- 🎯 Character Builder integration
- 🎯 VTT export (JSON for Roll20, Foundry)
- 🎯 User accounts (cloud save)

---

## Success Criteria

### Must Have (MVP)
- ✅ Support all 12+ spellcasting classes
- ✅ Accurate spell slot calculation
- ✅ Accurate prepared/known spell limits
- ✅ Spell selection with checkboxes
- ✅ LocalStorage persistence
- ✅ Mobile-responsive layout
- ✅ Loading and error states
- ✅ TDD test coverage

### Nice to Have (Stretch Goals)
- ⭐ Spell search/filter within generator
- ⭐ "Quick picks" suggestions
- ⭐ Spell level badge on cards
- ⭐ Animated transitions

---

## Implementation Notes

### Key Dependencies
- Existing `useEntityList` composable (reference only, not reused)
- Existing `SpellCard` component (or create lightweight version)
- NuxtUI components (`UAccordion`, `UCheckbox`, `USelectMenu`)
- LocalStorage API (browser native)

### Performance Considerations
- Debounce auto-save (500ms) to avoid excessive writes
- Lazy load spell cards (IntersectionObserver if >100 spells)
- Cache class data (useAsyncData with caching)

### Accessibility
- Keyboard navigation for checkboxes
- ARIA labels for selection counts
- Screen reader announcements for state changes
- Focus management in accordions

---

## Open Questions (Resolved)

1. **Q:** Should we fetch spells_known from API or hardcode?
   **A:** Hardcode for MVP (API field is null), can enhance later

2. **Q:** How to handle ability score modifiers?
   **A:** Default to +3 for MVP, add input field in Phase 2

3. **Q:** Single page or multi-step wizard?
   **A:** Single page (simpler, faster to build)

4. **Q:** LocalStorage or API persistence?
   **A:** LocalStorage for MVP (no backend changes needed)

---

## Approval

**Design Status:** ✅ Approved
**Ready for Implementation:** Yes
**Next Step:** Create worktree and implementation plan
