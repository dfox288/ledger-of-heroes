# D&D Rulebook Project - Claude Code Instructions

This is the coordination repository for the D&D Rulebook project. It contains centralized documentation and GitHub Issues for frontend/backend coordination.

## Project Architecture

```
dnd-rulebook-project/      (this repo - coordination hub)
├── docs/
│   ├── backend/           # Laravel backend documentation
│   └── frontend/          # Nuxt frontend documentation
├── README.md              # Issue workflow & labels
└── CLAUDE.md              # This file

dnd-rulebook-frontend/     (separate repo - Nuxt 3 + Nuxt UI)
dnd-rulebook-parser/       (separate repo - Laravel 11 API)
```

## Working in This Repository

This repo is for **documentation and coordination only**. No application code lives here.

### When to Use This Repo

- Creating/managing cross-cutting GitHub Issues
- Reading documentation about either frontend or backend
- Writing session handovers or implementation plans
- Reviewing API contracts and proposals

### Documentation Structure

```
docs/
├── backend/
│   ├── handovers/       # Session handover documents
│   ├── plans/           # Implementation plans
│   ├── proposals/       # API enhancement proposals
│   ├── reference/       # Stable reference docs (API, filters, search)
│   ├── archive/         # Old/completed handovers
│   └── DND-FEATURES.md  # D&D 5e feature roadmap
└── frontend/
    ├── handovers/       # Session handover documents
    ├── plans/           # Implementation plans
    ├── proposals/       # API enhancement proposals
    ├── reference/       # Component docs, filter guides
    └── archive/         # Old/completed handovers
```

## Key Reference Documents

### Backend
- `docs/backend/DND-FEATURES.md` - D&D 5e API features (tags, filtering, search)
- `docs/backend/reference/MEILISEARCH-FILTERS.md` - Filter syntax documentation
- `docs/backend/reference/SEARCH.md` - Search system documentation
- `docs/backend/reference/API-EXAMPLES.md` - API usage examples
- `docs/backend/plans/README.md` - Active implementation plans

### Frontend
- `docs/frontend/reference/COMPONENTS.md` - UI component library
- `docs/frontend/reference/FILTER-COMPOSABLES-MIGRATION-GUIDE.md` - Filter system
- `docs/frontend/reference/UI-FILTER-LAYOUT-GUIDE.md` - Layout patterns

## Session Handovers

When completing work on frontend or backend:

1. Write handover to appropriate docs folder:
   ```
   docs/backend/handovers/SESSION-HANDOVER-YYYY-MM-DD-HHMM-topic.md
   docs/frontend/handovers/SESSION-HANDOVER-YYYY-MM-DD-HHMM-topic.md
   ```

2. Include in handover:
   - What was accomplished
   - Files modified
   - Test status
   - Next steps / known issues

3. Update symlink in the sub-project (if applicable):
   ```bash
   # From dnd-rulebook-parser/
   ln -sf ../../dnd-rulebook-project/docs/backend/handovers/SESSION-HANDOVER-*.md docs/LATEST-HANDOVER.md

   # From dnd-rulebook-frontend/
   ln -sf ../../dnd-rulebook-project/docs/frontend/handovers/SESSION-HANDOVER-*.md docs/LATEST-HANDOVER.md
   ```

## GitHub Issues Workflow

### Creating Issues (Claude Agent Instructions)

When creating issues via `gh issue create`, follow these structured formats.

**Required labels for every issue:**
1. Assignee: `frontend`, `backend`, or `both`
2. Type: `bug`, `feature`, `api-contract`, `data-issue`, `refactor`, etc.
3. Source (if discovered during work): `from:frontend`, `from:backend`, `from:manual-testing`

#### Bug Report Format

```bash
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "bug: Brief description of the problem" \
  --label "backend,bug,from:backend" \
  --body "$(cat <<'EOF'
## Summary
One sentence describing the bug.

## Where
Endpoint, component, or user flow affected (e.g., `GET /api/v1/spells`, `SpellCard.vue`)

## Expected Behavior
What should happen.

## Actual Behavior
What happens instead.

## Steps to Reproduce
1. Step one
2. Step two
3. Step three

## Additional Context
Error messages, logs, or screenshots if applicable.
EOF
)"
```

#### Feature Request Format

```bash
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "feat: Brief description of feature" \
  --label "frontend,feature" \
  --body "$(cat <<'EOF'
## Summary
What do you want to add and why?

## Proposed Solution
How should this be implemented? Affected files, approach.

## Acceptance Criteria
- [ ] Criterion one
- [ ] Criterion two
- [ ] Criterion three

## Related Issues
#123, #456 (if applicable)
EOF
)"
```

#### API Contract Format

```bash
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "api: Endpoint change description" \
  --label "both,api-contract" \
  --body "$(cat <<'EOF'
## Endpoint
`GET /api/v1/endpoint`

## Current Response
\`\`\`json
{ "current": "structure" }
\`\`\`

## Requested Change
What needs to be added or changed.

## Frontend Use Case
Why this change is needed.

## Breaking Change?
Yes/No - describe impact if yes.
EOF
)"
```

#### Data Issue Format

```bash
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "data: Entity name - brief problem" \
  --label "backend,data-issue" \
  --body "$(cat <<'EOF'
## Entity
Type: Spell/Monster/Item/etc.
Name: Entity name

## Problem
What's wrong with the data.

## Source Reference
PHB p.XXX, XGE, SRD, etc.

## Expected Data
What the correct values should be.
EOF
)"
```

#### Refactor/Cleanup Format

```bash
gh issue create --repo dfox288/dnd-rulebook-project \
  --title "refactor: Brief description" \
  --label "frontend,refactor" \
  --body "$(cat <<'EOF'
## Summary
What needs to be refactored and why.

## Current State
Description of current implementation, affected files.

## Proposed Solution
How to improve it, code examples if helpful.

## Affected Files
- `path/to/file1.ts`
- `path/to/file2.vue`
EOF
)"
```

### Checking Issues

```bash
# Your inbox
gh issue list --repo dfox288/dnd-rulebook-project --label "backend" --state open
gh issue list --repo dfox288/dnd-rulebook-project --label "frontend" --state open

# All open
gh issue list --repo dfox288/dnd-rulebook-project --state open
```

### Labels Reference

| Category | Labels |
|----------|--------|
| Assignee | `frontend`, `backend`, `both` |
| Type | `bug`, `feature`, `api-contract`, `api`, `data-issue`, `performance`, `refactor`, `cleanup`, `testing`, `coverage`, `ux`, `breaking-change` |
| Priority | `priority:high`, `priority:medium` |
| Status | `blocked`, `needs-discussion` |
| Scope | `character`, `epic`, `tracking`, `quality` |
| Source | `from:manual-testing`, `from:frontend`, `from:backend` |

## D&D 5e API Features (Backend)

The backend API supports these D&D-specific features:

- **Universal Tags** - All 7 entities support Meilisearch tag filtering
- **Dual ID/Slug Routing** - `/api/v1/spells/123` or `/api/v1/spells/fireball`
- **Multi-Source Citations** - PHB, XGE, TCE, etc.
- **Advanced Filtering** - Range, logical operators, combined search+filter
- **30 Languages** - Including choice slots
- **Saving Throw Modifiers** - Advantage/disadvantage tracking
- **AC Modifier Categories** - ac_base, ac_bonus, ac_magic
- **Random Tables** - 54 tables (Wild Magic, etc.)
- **Monster Spell Syncing** - 129 spellcasters, 1,098 spell relationships
- **Global Search** - 7 entity types, typo-tolerant, <50ms

See `docs/backend/DND-FEATURES.md` for complete documentation.

## Entity Types

The API serves 7 main entity types:
1. **Spells** - 522+ spells with schools, components, damage types
2. **Monsters** - 400+ monsters with 12 type-specific strategies
3. **Items** - Equipment, magic items, potions, weapons, armor
4. **Races** - Player races with traits, languages, proficiencies
5. **Classes** - Character classes with subclasses, features, progressions
6. **Backgrounds** - Starting proficiencies, equipment, features
7. **Feats** - Prerequisites, modifiers, benefits

## Writing New Documentation

### Plans (for new features)
```markdown
# Feature Name Implementation Plan

**Created:** YYYY-MM-DD
**Estimated Effort:** X hours
**Branch:** (to be created)

## Overview
Brief description

## Phases
### Phase 1: Name
- BATCH 1.1: Task (time estimate)
  - Steps
  - Tests to write
  - Acceptance criteria

## Verification
- [ ] Tests pass
- [ ] Code quality gates
- [ ] API changes documented
```

### Handovers (session end)
```markdown
# Session Handover: Topic

**Date:** YYYY-MM-DD HH:MM
**Branch:** main (or feature branch)

## Completed
- What was done

## Files Changed
- List of files

## Test Status
- All tests passing (X tests)

## Next Steps
- What's left to do

## Known Issues
- Any blockers or concerns
```

## Best Practices

1. **Read before writing** - Check existing docs before creating new ones
2. **Archive old handovers** - Move to archive/ when superseded
3. **Keep reference docs current** - Update when API/components change
4. **Cross-reference** - Link related issues and docs
5. **Commit docs with code** - If docs are in sub-project, commit together
