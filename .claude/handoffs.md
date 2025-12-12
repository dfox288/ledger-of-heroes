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
**From:** backend | **Issue:** #537 | **Created:** 2025-12-12 21:30

HP modification endpoint is live. Replace client-side D&D calculations with backend calls.

**What I did:**
- Implemented `PATCH /api/v1/characters/{id}/hp` with full D&D 5e rule enforcement
- Handles damage (temp HP absorbs first), healing (caps at max), temp HP (higher-wins)
- Death saves auto-reset when healing from 0 HP

**What you need to do:**
- Update HP panel to call the new endpoint instead of local calculations
- Send `{"hp": "-X"}` for damage, `{"hp": "+X"}` for healing
- Send `{"temp_hp": X}` for temp HP (0 clears it)
- Update local state from the response

**Technical details:**
- Endpoint: `PATCH /api/v1/characters/{id}/hp`
- Request: `{"hp": "-12"}` or `{"hp": "+8"}` or `{"temp_hp": 10}`
- Response:
```json
{
  "data": {
    "current_hit_points": 39,
    "max_hit_points": 52,
    "temp_hit_points": 0,
    "death_save_successes": 0,
    "death_save_failures": 0
  }
}
```

**Test with:**
```bash
curl -X PATCH http://localhost:8080/api/v1/characters/1/hp \
  -H "Content-Type: application/json" \
  -d '{"hp": "-12"}'
```

**Related:**
- Follows from: #536 (backend implementation)
- See also: `app/Http/Controllers/Api/CharacterHpController.php`

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
