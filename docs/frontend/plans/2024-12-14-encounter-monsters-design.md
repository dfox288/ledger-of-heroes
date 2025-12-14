# DM Screen: Encounter Monsters Design

## Overview

Add monsters to the DM Screen initiative tracker, allowing DMs to:
- Build encounters by adding monsters from the compendium
- Quick-add monsters during combat
- Track monster HP with backend persistence
- Manage turn order with mixed party + monster initiative

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Workflow | Encounter builder + Quick add | Prep encounters beforehand, add reinforcements mid-combat |
| Persistence | Backend per party | Encounters survive browser refresh, accessible across devices |
| Data stored | Monster ref + HP only | Stats fetched from API, keeps backend simple |
| Instance naming | Auto-number + rename | "Goblin 1" by default, allow "Goblin Chief" rename |
| Scope | Party-scoped | Encounters belong to a party, viewed on that party's DM screen |
| Add UI | Search modal + quantity | Familiar pattern, works on mobile, doesn't take permanent space |

## Data Model

### Backend: `encounter_monsters` table

```
encounter_monsters
├── id
├── party_id (FK → parties)
├── monster_id (FK → monsters)
├── label (string, e.g., "Goblin 1" or "Goblin Chief")
├── current_hp (integer)
├── max_hp (integer, snapshot at creation)
├── created_at
└── updated_at
```

No separate "encounters" table - monsters belong directly to a party.

### Frontend: Extended combat state (localStorage)

```typescript
// Key format supports both characters and monsters
initiatives: Record<string, number>  // "char_1" or "monster_5" → initiative
```

## API Endpoints

### New endpoints

```
GET    /api/parties/{id}/monsters         → List monsters in party's encounter
POST   /api/parties/{id}/monsters         → Add monster(s) to encounter
PATCH  /api/parties/{id}/monsters/{id}    → Update instance (HP, label)
DELETE /api/parties/{id}/monsters/{id}    → Remove monster instance
DELETE /api/parties/{id}/monsters         → Clear all monsters
```

### POST request

```json
{
  "monster_id": 5,
  "quantity": 3
}
```

### Response shape

```json
{
  "id": 15,
  "monster_id": 42,
  "label": "Goblin 1",
  "current_hp": 7,
  "max_hp": 7,
  "monster": {
    "name": "Goblin",
    "slug": "mm:goblin",
    "armor_class": 15,
    "hit_points": { "average": 7, "formula": "2d6" },
    "speed": { "walk": 30 },
    "challenge_rating": "1/4",
    "actions": [
      {
        "name": "Scimitar",
        "attack_bonus": 4,
        "damage": "1d6+2 slashing",
        "reach": "5 ft."
      },
      {
        "name": "Shortbow",
        "attack_bonus": 4,
        "damage": "1d6+2 piercing",
        "range": "80/320 ft."
      }
    ]
  }
}
```

## UI Components

### Combat Table

- Monsters appear in same table as characters, sorted by initiative
- Visual distinction: monsters have red/orange accent (characters neutral/green)
- Monster rows show: Label, HP bar, AC, Initiative, Actions (condensed)
- Click row → expand to show full monster stat block

### Add Monster Modal

- Trigger: "Add Monster" button in combat toolbar
- Search field with type-ahead (existing monster search API)
- Results show: Name, CR, HP, AC
- Quantity selector (default: 1, max: 20)
- "Add" button → creates instances, closes modal

### Monster Row Actions

- Inline HP adjustment (click HP → +/- buttons or direct edit)
- Rename (click label → edit inline)
- Remove (trash icon with confirmation)

### Combat Toolbar

```
[Roll Initiative] [Start Combat] [Add Monster] [Next Turn] [Reset]
```

## Combat Integration

### Initiative tracking

- Monsters use existing `useDmScreenCombat` composable
- Key format: `monster_${instanceId}` vs `char_${characterId}`
- Initiative stored in localStorage alongside character initiatives
- Sorting treats both equally - all combatants in one order

### Turn order example

```
Initiative 18: Zephyr (Rogue)          ← character
Initiative 15: Goblin 1                ← monster
Initiative 15: Goblin 2                ← monster
Initiative 12: Aldric (Fighter)        ← character
Initiative 8:  Bugbear Chief           ← monster
```

### HP sync

- Monster HP changes immediately PATCH to backend
- Debounced (300ms) to avoid spam during rapid adjustments
- Optimistic UI - update locally first, sync in background

### Reset behavior

- "Reset Combat" clears initiatives but keeps monsters
- "Clear Encounter" removes all monsters (separate action)

## Implementation Phases

### Phase 1: Backend
- New `encounter_monsters` table + migration
- CRUD endpoints for party monsters
- Include nested monster data in response

### Phase 2: Frontend - Data Layer
- Nitro routes for new endpoints
- Extend `useDmScreenCombat` to handle monster keys
- Types for encounter monsters

### Phase 3: Frontend - UI
- Add Monster modal with search + quantity
- Monster rows in combat table (visual distinction)
- Inline HP editing with backend sync
- Label rename, remove actions

### Phase 4: Polish
- Monster stat block expansion (click row)
- Attack quick-reference display
- Clear encounter confirmation

## Out of Scope

- Encounter templates/presets
- XP/CR budget calculator
- Lair actions / legendary actions tracking
- Multi-party encounters
