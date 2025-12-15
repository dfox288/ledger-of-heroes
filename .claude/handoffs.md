# Agent Handoffs

<!--
This file enables async communication between Claude Code agents working in different repos.

HOW IT WORKS:
- When an agent creates cross-repo work, they append a handoff here with full context
- When an agent starts a session, they check for handoffs addressed to them
- After processing a handoff, DELETE that section from this file

LOCATION: wrapper/.claude/handoffs.md
-->

## Pending Handoffs

<!-- Agents: Add new handoffs below this line. Delete handoffs after processing. -->

## For: frontend
**From:** backend | **Issue:** #681 | **Created:** 2025-12-15 16:35

Encounter presets API is ready! DMs can now save and load monster groups.

**What I did:**
- Added CRUD endpoints for encounter presets at `/parties/{party}/encounter-presets`
- Load endpoint creates EncounterMonster records with proper auto-labeling
- Response includes `monster_name` and `challenge_rating` for easy display
- 20 tests covering all operations

**What you need to do:**
- Add "Save as Preset" UI when encounter has monsters
- Add "Load Preset" selector in DM Screen encounter panel
- Implement preset management (rename, delete)

**API contract:**
| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/parties/{party}/encounter-presets` | List presets |
| POST | `/parties/{party}/encounter-presets` | Create preset |
| PATCH | `/parties/{party}/encounter-presets/{id}` | Rename |
| DELETE | `/parties/{party}/encounter-presets/{id}` | Delete |
| POST | `/parties/{party}/encounter-presets/{id}/load` | Load into encounter |

**Response shape:**
```json
{
  "data": [{
    "id": 1,
    "name": "Goblin Patrol",
    "monsters": [
      { "monster_id": 5, "quantity": 4, "monster_name": "Goblin", "challenge_rating": "1/4" }
    ],
    "created_at": "...",
    "updated_at": "..."
  }]
}
```

**Test with:**
```bash
# Create preset
curl -X POST "http://localhost:8080/api/v1/parties/1/encounter-presets" \
  -H "Content-Type: application/json" \
  -d '{"name": "Test", "monsters": [{"monster_id": 1, "quantity": 2}]}'

# Load preset
curl -X POST "http://localhost:8080/api/v1/parties/1/encounter-presets/1/load"
```

**Related:**
- Closes: #667 (original request)
- Backend PR: dfox288/ledger-of-heroes-backend#182
- See also: `useEncounterMonsters.ts`

---

## For: backend
**From:** frontend | **Issue:** #647 | **Created:** 2025-12-15 08:50

Class resource counters needed for character sheet (Rage, Ki Points, Sorcery Points, etc.).

**What frontend needs:**
- Character response to include `class_resources` array
- Each resource: id, name, slug, current, max, reset_on
- PATCH endpoint to update current value
- Resources auto-reset on short/long rest

**Priority resources:**
| Class | Resource | Max | Resets |
|-------|----------|-----|--------|
| Barbarian | Rage | 2-6 by level | Long rest |
| Monk | Ki Points | = monk level | Short rest |
| Fighter | Action Surge | 1-2 | Short rest |
| Sorcerer | Sorcery Points | = sorc level | Long rest |

**Expected response shape:**
```json
{
  "class_resources": [
    { "id": 1, "slug": "rage", "name": "Rage", "current": 2, "max": 3, "reset_on": "long_rest" }
  ]
}
```

**Test with:**
```bash
# After implementation
curl "http://localhost:8080/api/v1/characters/1" | jq '.data.class_resources'
```

**Related:**
- Blocks: #632 (Frontend class resource counters)
- Also needed for: #606 (DM Screen resource tracker)

---

<!-- HANDOFF TEMPLATE (copy this when creating a new handoff):

## For: frontend|backend
**From:** backend|frontend | **Issue:** #N | **Created:** YYYY-MM-DD HH:MM

[Brief description]

**What I did:**
- [Key changes made]

**What you need to do:**
- [Specific tasks]

**Technical details:**
- Endpoint: `GET /api/v1/example`
- Request: `?filter=field=value`
- Response: `{ data: { id, name, slug } }`

**Test with:**
```bash
curl "http://localhost:8080/api/v1/example"
```

**Related:**
- Closes/Follows from: #N
- See also: `path/to/relevant/file`

---

-->
