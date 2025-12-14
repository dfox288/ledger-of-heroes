---
description: Write a handoff to the other repo after creating cross-repo work
---

# Write Handoff

After creating a GitHub issue that requires work in the other repo, write a handoff with full context.

## Why Handoffs Matter

GitHub issues capture WHAT needs to be done. Handoffs capture:
- HOW you implemented your side
- The exact API contract (endpoints, filters, response shapes)
- Working test commands
- Decisions made during implementation
- Gotchas and edge cases discovered

## Instructions

1. **Determine the target:** `frontend` or `backend`

2. **Open the handoff file:**
   ```bash
   # View current handoffs
   cat ../wrapper/.claude/handoffs.md
   ```

3. **Append your handoff** to `../wrapper/.claude/handoffs.md` using this template:

### For Backend → Frontend handoffs:

```markdown
## For: frontend
**From:** backend | **Issue:** #N | **Created:** YYYY-MM-DD HH:MM

[Brief description of what was implemented]

**What I did:**
- [Endpoints/models/services added]
- [Key implementation decisions]

**What frontend needs to do:**
- [UI components needed]
- [Pages to create]
- [Filters to implement]

**API contract:**
- Endpoint: `GET /api/v1/endpoint`
- Filters: `field` (equals), `tags` (IN), `is_active` (boolean)
- Response:
```json
{
  "data": [{ "id": 1, "name": "Example", "slug": "example" }],
  "meta": { "total": 100, "per_page": 24 }
}
```

**Test with:**
```bash
curl "http://localhost:8080/api/v1/endpoint?filter=field=value"
```

**Related:** Follows from #N, see `app/Http/Controllers/Api/Controller.php`

---
```

### For Frontend → Backend handoffs:

```markdown
## For: backend
**From:** frontend | **Issue:** #N | **Created:** YYYY-MM-DD HH:MM

[Brief description of what's needed or broken]

**Context:**
- [UI feature that needs this]
- [User flow that triggers this]

**Observed behavior:**
- [What currently happens]

**Expected behavior:**
- [What should happen]

**Reproduction:**
```bash
curl "http://localhost:8080/api/v1/endpoint"
# Returns: [actual response]
# Expected: [desired response]
```

**Frontend is blocked on:** [Component/page waiting for this]

**Related:** See `app/components/Example.vue`, `app/pages/example/index.vue`

---
```

4. **Commit the handoff** to the wrapper repo:
   ```bash
   cd ../wrapper && git add .claude/handoffs.md && git commit -m "handoff: [brief description] for #N"
   ```
