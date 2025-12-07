# D&D Rulebook Project Documentation

Centralized documentation for frontend and backend projects.

## Structure

```
docs/
├── frontend/        # Nuxt frontend documentation
│   ├── handovers/   # Session handovers
│   ├── plans/       # Implementation plans
│   ├── proposals/   # API enhancement proposals
│   ├── reference/   # Stable reference docs
│   └── archive/     # Old handovers
└── backend/         # Laravel backend documentation
    ├── handovers/   # Session handovers
    ├── plans/       # Implementation plans
    ├── proposals/   # API enhancement proposals
    ├── reference/   # Stable reference docs
    ├── archive/     # Old handovers
    └── DND-FEATURES.md  # D&D feature roadmap
```

## Why Centralized?

1. **Cross-cutting visibility** - See related frontend/backend docs together
2. **Reduces duplication** - Shared proposals live in one place
3. **Issue coordination** - GitHub Issues + docs in same repo
4. **Cleaner sub-repos** - Frontend/backend stay focused on code

## Writing New Docs

### From Frontend

```bash
# Plans
../dnd-rulebook-project/docs/frontend/plans/YYYY-MM-DD-feature-design.md

# Handovers
../dnd-rulebook-project/docs/frontend/handovers/SESSION-HANDOVER-YYYY-MM-DD-topic.md

# Then update the symlink in frontend/docs/
ln -sf ../../dnd-rulebook-project/docs/frontend/handovers/SESSION-HANDOVER-YYYY-MM-DD-topic.md docs/LATEST-HANDOVER.md
```

### From Backend

```bash
# Plans
../dnd-rulebook-project/docs/backend/plans/YYYY-MM-DD-feature-design.md

# Handovers
../dnd-rulebook-project/docs/backend/handovers/SESSION-HANDOVER-YYYY-MM-DD-topic.md

# Then update the symlink in importer/docs/
ln -sf ../../dnd-rulebook-project/docs/backend/handovers/SESSION-HANDOVER-YYYY-MM-DD-topic.md docs/LATEST-HANDOVER.md
```

## What Stays in Sub-Projects

Each sub-project keeps these files locally:

- `docs/PROJECT-STATUS.md` - Project-specific metrics
- `docs/LATEST-HANDOVER.md` - Symlink to latest handover here
- `docs/README.md` - Points to this documentation

## Commit Strategy

When writing new docs:

1. Write the doc in this repo (`dnd-rulebook-project/docs/...`)
2. Commit and push this repo
3. If you updated LATEST-HANDOVER.md symlink in sub-project, commit that too
