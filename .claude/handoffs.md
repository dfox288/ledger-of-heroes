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
**From:** backend | **Issue:** #747 | **Created:** 2025-12-18 15:30

Entity view endpoints now expose choice information for displaying fixed vs choice bonuses.

**What I did:**
- Added `choices` field to Race, Background, and Feat resources
- Added `equipment_choices` field to Background resource (grouped format)
- Subraces include parent choices in `inherited_data.choices`

**What you need to do:**
- Update `AbilityScoresCard.vue` to show fixed modifiers vs ability score choices
- Update `BackgroundDetailModal.vue` to show language/proficiency choices
- Update `FeatDetailModal.vue` to show spell/proficiency/ability choices
- Use `equipment_choices` for background equipment options (same format as class)

**API Response Format:**

`GET /api/v1/races/{id}` - `choices` field:
```json
{
  "data": {
    "modifiers": [{ "ability_code": "CHA", "value": 2 }],
    "choices": [
      {
        "id": 123,
        "choice_type": "ability_score",
        "choice_group": "ability_score_choice",
        "quantity": 2,
        "constraint": "different",
        "description": "Choose two different ability scores to increase by 1",
        "is_required": true
      }
    ]
  }
}
```

`GET /api/v1/backgrounds/{id}` - `choices` + `equipment_choices`:
```json
{
  "data": {
    "choices": [
      { "choice_type": "language", "quantity": 2, "description": "Choose two languages" }
    ],
    "equipment_choices": [
      {
        "choice_group": "weapon_choice",
        "quantity": 1,
        "options": [
          { "option": "a", "label": "Longsword", "items": [{ "slug": "longsword", "name": "Longsword" }] },
          { "option": "b", "label": "Shortsword", "items": [{ "slug": "shortsword", "name": "Shortsword" }] }
        ]
      }
    ]
  }
}
```

**Choice types by entity:**
| Entity | Choice Types |
|--------|--------------|
| Race | `ability_score`, `language`, `proficiency`, `spell` |
| Background | `language`, `proficiency` (equipment in separate field) |
| Feat | `spell`, `proficiency`, `ability_score` |

**Test with:**
```bash
# Half-Elf has ability score choices
curl "http://localhost:8080/api/v1/races/1" | jq '.data.choices'

# Backgrounds with language choices
curl "http://localhost:8080/api/v1/backgrounds/1" | jq '.data.choices'
```

**Related:**
- Closes: #747
- OpenAPI docs updated: http://localhost:8080/docs/api

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
