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
