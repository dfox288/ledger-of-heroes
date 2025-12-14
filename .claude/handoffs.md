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

## For: backend
**From:** frontend | **Issue:** #612 | **Created:** 2025-12-14 15:30

Spell slot tracking and preparation toggle endpoints needed for Character Sheet Spells Tab Phase 2.

**What I did:**
- Designed Spells Tab UI (see `wrapper/docs/frontend/plans/2025-12-14-spells-tab-design.md`)
- Created issue #612 with detailed implementation plan
- Added implementation comment with migration order and test checklist

**What you need to do:**
1. Add `character_spell_slots` table (or JSON column) for tracking spent slots
2. Implement `PATCH /characters/{id}/spell-slots/{level}` - action-based slot usage
3. Implement `PATCH /characters/{id}/spells/{spellId}` - preparation toggle
4. Update `POST /characters/{id}/long-rest` to reset spell slots

**Technical details:**
- Full plan: `wrapper/docs/frontend/plans/2025-12-14-backend-spell-slots-plan.md`
- Action-based API recommended: `{ "action": "use" | "restore" | "reset" }`
- Existing `GET /spell-slots` works great, just need mutation endpoints

**Test with:**
```bash
# Current (works)
curl "http://localhost:8080/api/v1/characters/flame-phoenix-t5kg/spell-slots"

# Needed
curl -X PATCH "http://localhost:8080/api/v1/characters/{id}/spell-slots/1" \
  -H "Content-Type: application/json" \
  -d '{"action": "use"}'
```

**Related:**
- Blocks Phase 2 of: #556 (Spells Tab)
- See also: #612 for full details

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
