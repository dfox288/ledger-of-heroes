# Wizard Spellbook UI - Phase 2 Design

**Issue:** #680
**Created:** 2025-12-15
**Status:** Design Complete

## Overview

Enhance the spell preparation experience for wizards with a two-column layout that makes it easy to manage their spellbook and prepare spells for the day.

**Goals:**
- Help wizards decide which spells to prepare (filtering by school, level, concentration, ritual)
- Make it fast to swap prepared spells (click-to-move between columns)

**Scope:** Wizards only (`preparation_method: 'spellbook'`). Other casters keep Phase 1 behavior.

## Layout

### Desktop (≥1024px)

```
┌─────────────────────────────────────────────────────────────┐
│  [Character Header]                                          │
├─────────────────────────────────────────────────────────────┤
│  Spellcasting Stats: DC 15 | Attack +7 | INT               │
│                                          Prepared: 8 / 12   │
├────────────────────────────┬────────────────────────────────┤
│  SPELLBOOK (24 spells)     │  PREPARED TODAY (8 spells)     │
│  ┌──────────────────────┐  │  ┌──────────────────────────┐  │
│  │ [🔍 Search...       ]│  │  │ 1st Level (3)            │  │
│  │ School: [All ▼]      │  │  │  • Mage Armor        ←   │  │
│  │ Level: [All ▼]       │  │  │  • Shield            ←   │  │
│  │ [□ Conc] [□ Ritual]  │  │  │  • Magic Missile     ←   │  │
│  └──────────────────────┘  │  │                          │  │
│                            │  │ 2nd Level (2)            │  │
│  1st Level (6)             │  │  • Misty Step        ←   │  │
│   • Charm Person       →   │  │  • Hold Person       ←   │  │
│   • Detect Magic       →   │  │                          │  │
│   • Sleep              →   │  │ 3rd Level (3)            │  │
│                            │  │  • Fireball          ←   │  │
│  2nd Level (4)             │  │  • Counterspell      ←   │  │
│   • Invisibility       →   │  │  • Fly               ←   │  │
│   ...                      │  └──────────────────────────┘  │
└────────────────────────────┴────────────────────────────────┘
```

- **Left column (Spellbook):** All spells in spellbook with filters
- **Right column (Prepared Today):** Currently prepared spells grouped by level
- **Click** a spell to move it to the other column
- **Counter** shows "Prepared: X / Y" prominently

### Mobile (<768px)

Tab-based layout showing one column at a time:

```
┌─────────────────────────────┐
│ [Spellbook] [Prepared 8/12] │  ← Tab bar
├─────────────────────────────┤
│ [🔍 Search...    ] [Filter] │
├─────────────────────────────┤
│  1st Level (6)              │
│   • Charm Person        [+] │
│   • Detect Magic        [+] │
│   • Sleep               [+] │
│  ...                        │
└─────────────────────────────┘
```

- Filters collapse into "Filter" button opening a sheet
- `[+]` / `[-]` buttons for clear tap targets

## Interaction

### Spell Card States

| State | Visual | Behavior |
|-------|--------|----------|
| Available (in spellbook) | Normal card, `→` indicator | Click to prepare |
| Prepared | Highlighted border, `←` indicator | Click to unprepare |
| At limit (unprepared) | Greyed out (opacity-40) | Click disabled, tooltip "At preparation limit" |
| Always prepared | Gold accent, "Always" badge | Cannot be unprepared, no click |
| Ritual (unprepared) | "Ritual" badge | Can cast without preparing, click still works |

### Counter Behavior

- Shows `Prepared: 8 / 12` in header area
- Updates instantly on click (optimistic UI)
- When at limit: counter turns amber/warning color
- When over limit (edge case): counter turns red

### Animation

- Spell card fades out of source column (150ms)
- Appears at correct position in target column (sorted by level, then name)
- Subtle slide-in animation

## Filters

Filters apply to the **Spellbook column only** (left side).

```
┌────────────────────────────────────────┐
│ [🔍 Search spells...                 ] │
│ School: [All ▼]  Level: [All ▼]        │
│ [□ Concentration only] [□ Ritual only] │
└────────────────────────────────────────┘
```

- **Search:** Filters by spell name (debounced 150ms)
- **School dropdown:** All, Abjuration, Conjuration, Divination, Enchantment, Evocation, Illusion, Necromancy, Transmutation
- **Level dropdown:** All, Cantrip, 1st, 2nd, ... 9th
- **Checkboxes:** Toggle concentration/ritual filters
- Filters combine with **AND** logic
- Filters persist during session, reset on page reload
- Show "No matching spells" if filters exclude everything

**No filters on Prepared column:** Prepared list is small and already grouped by level.

## Component Architecture

### New Components

| Component | Purpose |
|-----------|---------|
| `SpellbookView.vue` | Two-column layout container, manages filter state |
| `SpellbookColumn.vue` | Left column - all spells with filters |
| `PreparedColumn.vue` | Right column - prepared spells grouped by level |
| `SpellbookFilters.vue` | Search + dropdowns + checkboxes |
| `SpellbookCard.vue` | Compact spell card for spellbook view |

### Data Flow

```
spells.vue (page)
    │
    ├── preparation_method === 'spellbook'
    │       └── SpellbookView
    │             ├── SpellbookColumn (filters, unprepared spells)
    │             └── PreparedColumn (prepared spells)
    │
    └── preparation_method !== 'spellbook'
            └── [Existing Phase 1 layout]
```

### State Management

- **Filter state:** Local to `SpellbookView` (no Pinia needed)
- **Spell preparation:** Uses existing `characterPlayStateStore.toggleSpellPreparation()`
- **No new API calls** - uses existing endpoints

## Files to Create/Modify

| File | Change |
|------|--------|
| `app/components/character/sheet/SpellbookView.vue` | New - main container |
| `app/components/character/sheet/SpellbookColumn.vue` | New - left column |
| `app/components/character/sheet/PreparedColumn.vue` | New - right column |
| `app/components/character/sheet/SpellbookFilters.vue` | New - filter controls |
| `app/components/character/sheet/SpellbookCard.vue` | New - compact spell card |
| `app/pages/characters/[publicId]/spells.vue` | Conditionally render SpellbookView |

## Testing Strategy

### Unit Tests

| Component | Test Cases |
|-----------|------------|
| `SpellbookView` | Renders two-column for spellbook casters, falls back for others |
| `SpellbookFilters` | Search filters by name, dropdowns filter by school/level, checkboxes filter concentration/ritual, filters combine with AND |
| `SpellbookColumn` | Displays unprepared spells, respects filters, click calls prepare action, greyed out at limit |
| `PreparedColumn` | Groups by level, click calls unprepare action, always-prepared spells not clickable |
| `SpellbookCard` | Shows correct state indicators, disabled when at limit |

### Integration Tests

- Full flow: Filter → Click spell → Appears in prepared column → Counter updates
- At-limit behavior: Prepare up to limit → Next click blocked → Unprepare one → Can prepare again
- Mobile: Tab switching works, preparation works across tabs

### Manual Testing

- Create wizard character, add 15+ spells to spellbook
- Test preparation workflow at various limits
- Verify animations feel snappy
- Test on mobile viewport

## Dependencies

- **Phase 1 complete:** #676 - preparation_method detection working
- **Backend:** No changes needed - uses existing endpoints

## Out of Scope (Future)

- Copy scroll feature (add spells to spellbook)
- Spell descriptions in expanded view
- Drag-and-drop (click-to-move is sufficient)
