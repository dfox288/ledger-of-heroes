# D&D Rulebook Project

Central coordination hub for the D&D 5e Rulebook application. This repository manages cross-cutting issues, API contracts, and shared documentation between frontend and backend.

## Project Overview

A comprehensive D&D 5th Edition digital rulebook with:
- **522+ spells** with advanced filtering and search
- **400+ monsters** with 12 type-specific AI strategies
- **7 searchable entity types**: Spells, Monsters, Items, Races, Classes, Backgrounds, Feats
- **Meilisearch-powered search** with typo tolerance (<50ms response)
- **Universal tag system** for custom categorization

## Repository Structure

```
dnd-rulebook-project/          ← You are here (coordination)
├── docs/
│   ├── backend/               # Laravel API documentation
│   │   ├── handovers/         # Session handovers
│   │   ├── plans/             # Implementation plans
│   │   ├── proposals/         # API enhancement proposals
│   │   ├── reference/         # API docs, filter syntax
│   │   └── DND-FEATURES.md    # D&D 5e feature roadmap
│   └── frontend/              # Nuxt frontend documentation
│       ├── handovers/         # Session handovers
│       ├── plans/             # Implementation plans
│       ├── proposals/         # Feature proposals
│       └── reference/         # Component docs, guides
├── README.md                  # This file
└── CLAUDE.md                  # Claude Code instructions

dnd-rulebook-frontend/         # Nuxt 3 + Nuxt UI frontend
dnd-rulebook-parser/           # Laravel 11 REST API
```

## Quick Links

| Resource | Link |
|----------|------|
| Frontend Repo | [dnd-rulebook-frontend](https://github.com/dfox288/dnd-rulebook-frontend) |
| Backend Repo | [dnd-rulebook-parser](https://github.com/dfox288/dnd-rulebook-parser) |
| All Issues | [View Issues](https://github.com/dfox288/dnd-rulebook-project/issues) |
| API Features | [DND-FEATURES.md](docs/backend/DND-FEATURES.md) |

## Documentation

### Key Reference Docs

**Backend (Laravel API)**
- [D&D Features Roadmap](docs/backend/DND-FEATURES.md) - Tags, filtering, search, D&D mechanics
- [Meilisearch Filters](docs/backend/reference/MEILISEARCH-FILTERS.md) - Filter syntax documentation
- [Search System](docs/backend/reference/SEARCH.md) - Global and per-entity search
- [API Examples](docs/backend/reference/API-EXAMPLES.md) - Request/response examples
- [Implementation Plans](docs/backend/plans/README.md) - Active development plans

**Frontend (Nuxt)**
- [Components](docs/frontend/reference/COMPONENTS.md) - UI component library
- [Filter Composables](docs/frontend/reference/FILTER-COMPOSABLES-MIGRATION-GUIDE.md) - Filter system
- [UI Layout Guide](docs/frontend/reference/UI-FILTER-LAYOUT-GUIDE.md) - Layout patterns

### Session Handovers

Latest development session notes are in:
- `docs/backend/handovers/` - Backend session handovers
- `docs/frontend/handovers/` - Frontend session handovers

---

## Issue Management

### Creating Issues

**From browser:** Click "New Issue" above

**From CLI:**
```bash
# Backend bug
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "Spell components not returned from API" \
  --label "backend,bug,from:manual-testing"

# Frontend feature
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "Add pagination to spell list" \
  --label "frontend,feature"

# API contract discussion
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "Spell filtering contract mismatch" \
  --label "both,api-contract" \
  --body "Frontend expects X, backend returns Y"
```

### Checking Your Inbox

```bash
# Backend team
gh issue list --repo dfox288/dnd-rulebook-project --label "backend" --state open

# Frontend team
gh issue list --repo dfox288/dnd-rulebook-project --label "frontend" --state open

# All open issues
gh issue list --repo dfox288/dnd-rulebook-project --state open
```

### Resolving Issues

```bash
gh issue close 42 --repo dfox288/dnd-rulebook-project \
  --comment "Fixed in frontend PR #123"
```

---

## Label System

### Assignee (who should handle it)
| Label | Description |
|-------|-------------|
| `frontend` | Frontend team should handle |
| `backend` | Backend team should handle |
| `both` | Requires coordination between teams |

### Type (what kind of issue)
| Label | Description |
|-------|-------------|
| `bug` | Something isn't working correctly |
| `feature` | New feature or enhancement |
| `api-contract` | API design discussion needed |
| `data-issue` | Missing or incorrect D&D data |
| `performance` | Performance improvement needed |

### Status
| Label | Description |
|-------|-------------|
| `blocked` | Waiting on external dependency |
| `needs-discussion` | Needs team input before proceeding |

### Source (who discovered it)
| Label | Description |
|-------|-------------|
| `from:manual-testing` | Found during manual testing |
| `from:frontend` | Discovered by frontend development |
| `from:backend` | Discovered by backend development |

---

## Workflow

1. **Discover issue** during development, testing, or code review
2. **Create issue** with appropriate labels (assignee + type + source)
3. **Assignee checks inbox** at session start
4. **Work on issue**, reference in commits/PRs
5. **Close with comment** explaining resolution

---

## Issue Templates

### Bug Report
```markdown
**What's broken:**
[Description of the problem]

**Where:**
[Endpoint, component, or user flow]

**Expected:**
[What should happen]

**Actual:**
[What happens instead]

**Steps to reproduce:**
1. ...
2. ...
```

### API Contract Request
```markdown
**Endpoint:** `GET /api/v1/...`

**Current response:**
```json
{ ... }
```

**Requested change:**
[What you need added/changed]

**Why:**
[Frontend use case that requires this]
```

### Data Issue
```markdown
**Entity:** [Spell/Monster/Item/etc.]

**Problem:**
[Missing data, incorrect values, etc.]

**Source reference:**
[PHB p.XXX, XGE, etc.]

**Expected data:**
[What the correct value should be]
```

---

## Why Centralized Documentation?

1. **Cross-cutting visibility** - See related frontend/backend docs together
2. **Reduces duplication** - Shared proposals and contracts in one place
3. **Issue coordination** - GitHub Issues + documentation in same repo
4. **Cleaner code repos** - Frontend/backend stay focused on code

---

## Contributing

When writing new documentation:

1. Write doc in appropriate `docs/` subdirectory
2. Follow naming conventions:
   - Plans: `YYYY-MM-DD-feature-name.md`
   - Handovers: `SESSION-HANDOVER-YYYY-MM-DD-HHMM-topic.md`
3. Commit and push this repo
4. Update symlinks in sub-projects if needed

See [CLAUDE.md](CLAUDE.md) for detailed Claude Code instructions.
