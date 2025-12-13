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
**From:** backend | **Issue:** #560 | **Created:** 2025-12-13 15:30

Party System API is complete. Frontend needs UI for party management and DM dashboard.

**What I did:**
- Created Party model with user ownership
- 8 API endpoints for full CRUD + character management + stats
- DM Dashboard stats endpoint with HP, AC, passives, saves, conditions, spell slots
- 37 tests covering all functionality

**What you need to do:**
- Party list page (`/parties`) - cards with name, description, character count
- Party detail page (`/parties/{id}`) - edit party, manage characters
- DM Dashboard (`/parties/{id}/dashboard`) - side-by-side character stats
- Add character modal - search/select from DM's characters

**API Endpoints:**
| Method | Endpoint | Purpose |
|--------|----------|---------|
| POST | `/api/v1/parties` | Create party `{ name, description? }` |
| GET | `/api/v1/parties` | List DM's parties |
| GET | `/api/v1/parties/{id}` | Get party with characters |
| PUT | `/api/v1/parties/{id}` | Update party |
| DELETE | `/api/v1/parties/{id}` | Delete party |
| POST | `/api/v1/parties/{id}/characters` | Add character `{ character_id }` |
| DELETE | `/api/v1/parties/{id}/characters/{characterId}` | Remove character |
| GET | `/api/v1/parties/{id}/stats` | DM Dashboard data |

**Stats response shape:**
```json
{
  "data": {
    "party": { "id": 1, "name": "..." },
    "characters": [{
      "id": 1, "public_id": "...", "name": "...",
      "level": 5, "class_name": "Fighter",
      "hit_points": { "current": 38, "max": 45, "temp": 0 },
      "armor_class": 18, "proficiency_bonus": 3,
      "passive_skills": { "perception": 14, "investigation": 10, "insight": 12 },
      "saving_throws": { "STR": 5, "DEX": 2, "CON": 4, "INT": 0, "WIS": 1, "CHA": -1 },
      "conditions": [{ "name": "Poisoned", "slug": "poisoned", "level": null }],
      "spell_slots": { "1": { "current": 2, "max": 4 } }
    }]
  }
}
```

**Test with:**
```bash
# Create party
curl -X POST "http://localhost:8080/api/v1/parties" -H "Authorization: Bearer $TOKEN" -d '{"name":"Test Party"}'

# Get stats
curl "http://localhost:8080/api/v1/parties/1/stats" -H "Authorization: Bearer $TOKEN"
```

**Related:**
- Follows from: #559 (backend complete)
- See also: `app/Http/Controllers/Api/PartyController.php`

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
