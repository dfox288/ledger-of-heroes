# Prepared Caster Spell Selection

**Issue:** #723
**Created:** 2025-12-16
**Status:** Design Complete

## Overview

Add a "Prepare Spells" mode to the character spells page for clerics, druids, and paladins. These "prepared" casters choose spells daily from their **full class spell list**, unlike wizards who prepare from a limited spellbook.

## User Flow

1. User visits `/characters/{id}/spells` for a cleric/druid/paladin
2. Sees current view: stats bar with "X / Y Prepared" counter
3. **New:** "Prepare Spells" button next to counter (only shown when `preparation_method === 'prepared'`)
4. Clicking toggles to "Prepare Spells" mode:
   - Stats bar stays visible (counter updates in real-time)
   - Spell list replaced with full class spell list
   - Filters appear: search box, level dropdown, "hide prepared" toggle
   - Default filter: max castable level for this class
5. User toggles spells using existing prepare checkbox on SpellCard
6. "Back to My Spells" button returns to normal view

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| UI location | Inline on spells page | Contextual, no extra navigation |
| View switching | Replace spell list (toggle mode) | Full class list is 100+ spells; side-by-side would overwhelm |
| Filters | Essential only (search, level, hide-prepared) | Keep simple, can add more later |
| Spell display | Reuse existing SpellCard accordion | Already has prepare toggle, consistent UX |
| Always-prepared | Show with disabled toggle | Complete picture of available spells |
| Default filter | Max level = character's highest castable | Reduces initial list, shows relevant spells |
| Entry point | Button in stats bar next to counter | Natural placement near preparation info |

## Components

### Reused (no changes needed)

| Component | Purpose |
|-----------|---------|
| `SpellCard` | Display spells with prepare toggle |
| `characterPlayState` store | `toggleSpellPreparation` action |

### New Components

| Component | Purpose |
|-----------|---------|
| `PrepareSpellsView.vue` | Container for selection mode, fetches available spells |
| `PrepareSpellsFilters.vue` | Search input + level dropdown + hide-prepared toggle |

### Modified

| Component | Changes |
|-----------|---------|
| `spells.vue` | Add mode toggle, conditionally render PrepareSpellsView |
| Stats bar section | Add "Prepare Spells" button for prepared casters |

## API

**Endpoint:** `GET /api/characters/{id}/available-spells`

**Query params:**
- `max_level` - Filter by max spell level (default: character's max castable)
- `class` - Filter by class slug (for multiclass)
- `include_known` - Include already-known spells (always true for this view)

**Response:** `{ data: SpellResource[] }`

Already exists and is proxied via Nitro.

## Data Flow

```
spells.vue
├── isPrepareMode (ref)
├── when !isPrepareMode: existing spell list view
└── when isPrepareMode:
    └── PrepareSpellsView
        ├── fetches /available-spells?max_level={n}&class={slug}
        ├── PrepareSpellsFilters (search, level, hide-prepared)
        └── SpellCard for each spell
            └── prepare toggle → store.toggleSpellPreparation()
```

## Edge Cases

1. **At preparation limit** - Disable prepare toggle, show message
2. **Always-prepared spells** - Show in list with disabled toggle, marked as "Always Prepared"
3. **Multiclass** - Filter by class slug, show class-specific spell list
4. **No spells at selected level** - Empty state message

## Out of Scope (v1)

- Additional filters (school, concentration, ritual, components)
- Pagination (level filter should keep list manageable)
- Spell search across all levels simultaneously
