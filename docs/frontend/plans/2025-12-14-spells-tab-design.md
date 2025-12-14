# Spells Tab Design

**Issue:** #556 - Character Sheet: Spells Tab
**Created:** 2025-12-14
**Status:** Approved

## Overview

Comprehensive spell management page for spellcasting characters. Phase 1 delivers display functionality, Phase 2 adds interactive casting/preparation features (requires backend work).

## Scope

### Phase 1 (This Issue) - Display Only
- Dedicated spells page at `/characters/{publicId}/spells`
- Spellcasting stats bar (DC, Attack, Ability)
- Spell slots with crystal icons
- Cantrips section (separated)
- Leveled spells grouped by level
- Expandable spell cards with full details
- Prepared/unprepared visual indicators
- Upcast indicator badge

### Phase 2 (Future - Needs Backend)
- Toggle spell preparation
- Cast spell (consume slot)
- Upcast modal (choose slot level)
- Clickable slot crystals to track usage
- Reset slots on long rest

## Page Structure

**Route:** `/characters/[publicId]/spells.vue`

```
┌─────────────────────────────────────────────────┐
│  CharacterPageHeader                            │
│  (back button, play mode, portrait, actions)    │
├─────────────────────────────────────────────────┤
│  Spellcasting Stats Bar                         │
│  [Spell DC: 15] [Attack: +7] [Ability: WIS]     │
│  Prepared: 8/12                                 │
├─────────────────────────────────────────────────┤
│  Spell Slots (crystal icons per level)          │
│  1st: 💎💎💎◇  2nd: 💎💎◇  3rd: 💎◇            │
├─────────────────────────────────────────────────┤
│  Cantrips Section                               │
│  [expandable spell cards]                       │
├─────────────────────────────────────────────────┤
│  1st Level Spells                               │
│  [expandable spell cards]                       │
├─────────────────────────────────────────────────┤
│  2nd Level Spells...                            │
└─────────────────────────────────────────────────┘
```

## Components

### SpellCard (New)

**File:** `app/components/character/sheet/SpellCard.vue`

**Collapsed state:**
- Spell name (bold)
- Level indicator (cantrip / 1st / 2nd / etc.)
- Badges: `Concentration`, `Ritual`, `Scalable` (if upcastable)
- Prepared indicator: ✓ checkmark for prepared, dimmed for unprepared
- "Always prepared" indicator (lock icon or "Always" badge)

**Expanded state (click to toggle):**
- Casting time
- Range
- Components (V, S, M with material details)
- Duration
- Full description text
- "At Higher Levels" section (if upcastable)

**Visual treatment:**
- Prepared spells: normal opacity, accent border
- Unprepared spells: slightly dimmed (opacity-60)
- Always-prepared: checkmark + lock icon

**Future-ready:** Accepts optional `onCast` prop for casting button (disabled until Phase 2).

### SpellSlots (Update Existing)

**File:** `app/components/character/sheet/SpellSlots.vue`

**Changes:**
- Replace circles with `i-game-icons-crystal-shine` icons
- Size: `w-7 h-7` (28px) for better visibility/tap targets
- Available: `text-spell-500 dark:text-spell-400`
- Used: `text-gray-300 dark:text-gray-600`

**Warlock Pact Magic:**
- Separate "Pact Slots" section with same icon style

### SpellsPanel (Update Existing)

**File:** `app/components/character/sheet/SpellsPanel.vue`

**Changes:**
- Use new `SpellCard` component instead of badges
- Pass through expandable card props

## Data Flow

```typescript
// Page fetches (parallel)
const { data: character } = await useAsyncData(`spells-character-${publicId}`, ...)
const { data: spells } = await useAsyncData(`spells-data-${publicId}`, ...)
const { data: stats } = await useAsyncData(`spells-stats-${publicId}`, ...)

// Computed
const cantrips = computed(() => spells.filter(s => s.spell?.level === 0))
const leveledSpells = computed(() => spells.filter(s => s.spell?.level > 0))
const groupedByLevel = computed(() => groupBy(leveledSpells, s => s.spell.level))
```

## Edge Cases

| Case | Handling |
|------|----------|
| Non-spellcaster accessing page | Show "This character cannot cast spells" + back link |
| Spellcaster with no spells | Show stats bar + slots + empty state message |
| Dangling spell references | Filter out where `spell === null` |
| Warlock pact magic | Separate "Pact Slots" section |
| Multiclass with multiple casting | Combined spell list, separate slot pools if applicable |

## API Endpoints Used (Phase 1)

All existing:
- `GET /api/characters/{id}` - Character data
- `GET /api/characters/{id}/spells` - Spell list
- `GET /api/characters/{id}/stats` - Spellcasting stats, slots

## API Endpoints Needed (Phase 2)

**Toggle Preparation:**
```
PATCH /api/v1/characters/{id}/spells/{spellId}
Body: { "is_prepared": true/false }
```

**Spell Slot Tracking:**
```
GET /api/v1/characters/{id}/spell-slots
Response: { "slots": { "1": { "max": 4, "current": 3 }, ... } }

PATCH /api/v1/characters/{id}/spell-slots/{level}
Body: { "current": 2 }
```

**Long Rest Integration:**
- Existing `POST /characters/{id}/long-rest` should reset spell slots

## Testing Strategy

**Unit tests:**
- SpellCard renders collapsed/expanded states
- SpellCard shows correct badges (concentration, ritual, scalable)
- SpellCard handles prepared/unprepared/always-prepared states
- SpellSlots renders crystal icons correctly
- Page filters cantrips vs leveled spells
- Page handles non-spellcaster gracefully

**Integration tests:**
- Full page loads with MSW mocked data
- Spell card expansion works
- Empty state displays correctly

## Implementation Order

1. Create `SpellCard.vue` component with tests
2. Update `SpellSlots.vue` with crystal icons
3. Create `spells.vue` page following Features pattern
4. Add navigation link (conditional on isSpellcaster)
5. Update `SpellsPanel.vue` to use new card (for Overview tab)
