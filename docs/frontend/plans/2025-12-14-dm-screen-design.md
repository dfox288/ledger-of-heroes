# DM Screen UI Design

**Issue:** #564
**Date:** 2025-12-14
**Status:** Approved

## Overview

A dedicated DM Screen page for viewing party stats during gameplay. Serves two purposes:
1. **Combat tracker** - Quick reference for HP, AC, initiative, passives
2. **Session overview** - Party capabilities, gaps, and utility coverage

## Route

`/parties/[id]/dm-screen` - Separate page linked from party detail.

## Page Layout

```
┌─────────────────────────────────────────────────────────┐
│  Header: Party Name + Back Link + Refresh Button        │
├─────────────────────────────────────────────────────────┤
│  Party Summary (collapsible)                            │
│  [Languages] [Darkvision] [Healers] [Utility Spells]    │
├─────────────────────────────────────────────────────────┤
│  Combat Reference Table                                 │
│  Name | HP Bar | AC | Init | Perc | Inv | Ins           │
├─────────────────────────────────────────────────────────┤
│  Character Detail Cards (expand on row click)           │
└─────────────────────────────────────────────────────────┘
```

## Party Summary Panel

Collapsible panel at top, defaults collapsed on return visits (localStorage).

Four horizontal info blocks:

| Block | Content |
|-------|---------|
| **Languages** | List with "+N more" expansion |
| **Darkvision** | Count + warning for characters WITHOUT it |
| **Healers** | List of healers, or warning if none |
| **Utility** | Checkmarks for detect/dispel magic, counterspell |

Visual emphasis on gaps (warnings) and coverage (checkmarks).

## Combat Reference Table

Always-visible table for mid-combat scanning.

| Column | Content | Notes |
|--------|---------|-------|
| Name | Name + class indicator | Click to expand details |
| HP | Visual bar + numbers | Color: green→yellow→red |
| AC | Number badge | Highlight if 17+ |
| Init | Modifier with sign | For initiative calls |
| Perc | Passive Perception | Most used |
| Inv | Passive Investigation | Traps/hidden |
| Ins | Passive Insight | Social |

**Interactions:**
- Click row to expand character detail card
- Sortable by any column
- Visual states for low HP, active conditions, unconscious

## Character Detail Cards

Expandable below table row. Collapsible sub-sections:

### Combat Section (default open)
- Death saves: Visual circles for successes/failures
- Speeds: Walk, Fly, Swim, Climb
- Concentration status
- Saving throws: All six abilities

### Equipment Section
- Armor: Name, type, AC, stealth disadvantage
- Weapons: Name, damage, range
- Shield: Yes/No

### Capabilities Section
- Languages: Full list
- Tool proficiencies: Full list
- Size

### Spell Slots Section (spellcasters only)
- Visual slots per level: ●●●○ format

### Conditions Section
- Active conditions as badges
- Tooltips with effect descriptions

## File Structure

```
app/
├── pages/parties/[id]/dm-screen.vue
├── components/dm-screen/
│   ├── PartySummary.vue
│   ├── CombatTable.vue
│   ├── CombatTableRow.vue
│   ├── CharacterDetail.vue
│   ├── HpBar.vue
│   ├── DeathSaves.vue
│   └── SpellSlots.vue
├── types/dm-screen.ts
server/api/parties/[id]/
    └── stats.get.ts
```

## API

**Endpoint:** `GET /api/v1/parties/{id}/stats`

Proxied through Nitro at `/api/parties/{id}/stats`.

## Future Enhancements (Out of Scope)

- Add enemies to combat tracker
- Drag/drop initiative ordering
- Real-time updates via WebSocket
- Editable HP/conditions from DM Screen

## Testing

- Unit tests for each component
- Page test with MSW mock for stats endpoint
- Accessibility: keyboard navigation, screen reader labels
