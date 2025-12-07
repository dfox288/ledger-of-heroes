# Documentation Migration to Wrapper Project

**Date:** 2025-12-07
**Status:** Approved
**Scope:** Frontend + Backend docs → dnd-rulebook-project

---

## Overview

Migrate all documentation (plans, handovers, proposals, reference) from frontend and backend repos to the central `dnd-rulebook-project` wrapper repository. This centralizes cross-cutting documentation and reduces duplication.

## Decision Summary

| Decision | Choice |
|----------|--------|
| Archive handling | Flatten to single `archive/` folder per project |
| Post-migration | Delete from source (true move, not copy) |
| Handovers | Move all, update symlinks to point to wrapper |
| Approach | Script-based migration with verification |

## Current State

| Project | Location | File Count |
|---------|----------|------------|
| Frontend | `frontend/docs/` | ~272 .md files |
| Backend | `importer/docs/` | ~260 .md files |
| Wrapper | `dnd-rulebook-project/` | 1 file (README.md) |

## Target Structure

### Wrapper Repository (dnd-rulebook-project)

```
dnd-rulebook-project/
├── README.md                           # Existing issue coordination
├── docs/
│   ├── README.md                       # Explains doc organization
│   ├── frontend/
│   │   ├── handovers/                  # All session handovers
│   │   ├── archive/                    # Flattened old handovers
│   │   ├── plans/                      # Implementation plans
│   │   ├── proposals/                  # API enhancement proposals
│   │   └── reference/                  # Stable reference docs
│   └── backend/
│       ├── handovers/                  # Backend session handovers
│       ├── archive/                    # Flattened old handovers
│       ├── plans/                      # Backend implementation plans
│       ├── proposals/                  # Backend proposals
│       ├── reference/                  # Backend reference docs
│       └── DND-FEATURES.md             # D&D feature roadmap
```

### Sub-Projects (What Stays)

```
frontend/docs/
├── PROJECT-STATUS.md      # Project-specific metrics (stays)
├── LATEST-HANDOVER.md     # Symlink → wrapper handover
└── README.md              # Points to wrapper for docs

importer/docs/
├── PROJECT-STATUS.md      # Project-specific metrics (stays)
├── LATEST-HANDOVER.md     # Symlink → wrapper handover
└── README.md              # Points to wrapper for docs
```

## Migration Script

```bash
#!/bin/bash
# migrate-docs.sh
# Run from frontend/ directory

set -e  # Exit on error

WRAPPER="../dnd-rulebook-project"

echo "=== Phase 1: Create directory structure ==="
mkdir -p "$WRAPPER/docs/frontend"/{handovers,archive,plans,proposals,reference}
mkdir -p "$WRAPPER/docs/backend"/{handovers,archive,plans,proposals,reference}

echo "=== Phase 2: Migrate Frontend Docs ==="

# Current handovers
cp -p docs/handovers/*.md "$WRAPPER/docs/frontend/handovers/" 2>/dev/null || true

# Archived handovers (flatten nested structure)
find docs/archive/handovers -name "*.md" -exec cp -p {} "$WRAPPER/docs/frontend/archive/" \; 2>/dev/null || true

# Plans
cp -p docs/plans/*.md "$WRAPPER/docs/frontend/plans/" 2>/dev/null || true

# Proposals
cp -p docs/proposals/*.md "$WRAPPER/docs/frontend/proposals/" 2>/dev/null || true

# Reference
cp -p docs/reference/*.md "$WRAPPER/docs/frontend/reference/" 2>/dev/null || true

echo "=== Phase 3: Migrate Backend Docs ==="
cd ../importer

# Current handovers
cp -p docs/handovers/*.md "$WRAPPER/docs/backend/handovers/" 2>/dev/null || true

# Archived handovers (flatten)
find docs/archive/handovers -name "*.md" -exec cp -p {} "$WRAPPER/docs/backend/archive/" \; 2>/dev/null || true

# Plans
cp -p docs/plans/*.md "$WRAPPER/docs/backend/plans/" 2>/dev/null || true

# Proposals
cp -p docs/proposals/*.md "$WRAPPER/docs/backend/proposals/" 2>/dev/null || true

# Reference
cp -p docs/reference/*.md "$WRAPPER/docs/backend/reference/" 2>/dev/null || true

# DND-FEATURES.md (special file)
cp -p docs/DND-FEATURES.md "$WRAPPER/docs/backend/" 2>/dev/null || true

cd ../frontend

echo "=== Phase 4: Verify Migration ==="

# Count frontend files
FE_SOURCE=$(find docs/handovers docs/archive docs/plans docs/proposals docs/reference -name "*.md" 2>/dev/null | wc -l | tr -d ' ')
FE_DEST=$(find "$WRAPPER/docs/frontend" -name "*.md" | wc -l | tr -d ' ')

echo "Frontend: $FE_SOURCE source → $FE_DEST destination"

# Count backend files
cd ../importer
BE_SOURCE=$(find docs/handovers docs/archive docs/plans docs/proposals docs/reference docs/DND-FEATURES.md -name "*.md" 2>/dev/null | wc -l | tr -d ' ')
BE_DEST=$(find "$WRAPPER/docs/backend" -name "*.md" | wc -l | tr -d ' ')

echo "Backend: $BE_SOURCE source → $BE_DEST destination"

cd ../frontend

echo ""
echo "=== Verification Complete ==="
echo "Review the counts above. If they match, run Phase 5 (cleanup)."
echo ""
echo "To cleanup, run:"
echo "  # Frontend"
echo "  rm -rf docs/handovers docs/archive docs/plans docs/proposals docs/reference"
echo "  # Backend"
echo "  cd ../importer && rm -rf docs/handovers docs/archive docs/plans docs/proposals docs/reference docs/DND-FEATURES.md"
```

## Post-Migration Tasks

### 1. Update LATEST-HANDOVER.md Symlinks

```bash
# Frontend
cd frontend
rm docs/LATEST-HANDOVER.md
ln -s ../../dnd-rulebook-project/docs/frontend/handovers/SESSION-HANDOVER-2025-12-06-unified-choice-planning.md docs/LATEST-HANDOVER.md

# Backend
cd ../importer
rm docs/LATEST-HANDOVER.md
ln -s ../../dnd-rulebook-project/docs/backend/handovers/SESSION-HANDOVER-2025-12-03-1430-equipment-choice-items.md docs/LATEST-HANDOVER.md
```

### 2. Update Frontend CLAUDE.md

Add/update documentation section:

```markdown
## Documentation Locations

**Project docs live in the wrapper repo: `../dnd-rulebook-project/docs/frontend/`**

| Doc Type | Location |
|----------|----------|
| **Plans** | `../dnd-rulebook-project/docs/frontend/plans/` |
| **Handovers** | `../dnd-rulebook-project/docs/frontend/handovers/` |
| **Proposals** | `../dnd-rulebook-project/docs/frontend/proposals/` |
| **Reference** | `../dnd-rulebook-project/docs/frontend/reference/` |
| **Archive** | `../dnd-rulebook-project/docs/frontend/archive/` |

**Stays local:**
- `docs/PROJECT-STATUS.md` - This project's metrics
- `docs/LATEST-HANDOVER.md` - Symlink to latest handover
- `docs/README.md` - Points to wrapper docs

### Session Handover Workflow

1. Write handover: `../dnd-rulebook-project/docs/frontend/handovers/SESSION-HANDOVER-YYYY-MM-DD-topic.md`
2. Update symlink:
   ```bash
   ln -sf ../../dnd-rulebook-project/docs/frontend/handovers/SESSION-HANDOVER-YYYY-MM-DD-topic.md docs/LATEST-HANDOVER.md
   ```
3. Commit to BOTH repos (wrapper: handover file, frontend: symlink update)

### Writing Plans

Plans go in the wrapper repo:
```bash
../dnd-rulebook-project/docs/frontend/plans/YYYY-MM-DD-feature-design.md
```
```

### 3. Update Backend CLAUDE.md

Same pattern, pointing to `../dnd-rulebook-project/docs/backend/`

### 4. Create Wrapper docs/README.md

```markdown
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
    └── ...          # Same structure
```

## Why Centralized?

1. **Cross-cutting visibility** - See related frontend/backend docs together
2. **Reduces duplication** - Shared proposals live in one place
3. **Issue coordination** - GitHub Issues + docs in same repo
4. **Cleaner sub-repos** - Frontend/backend stay focused on code
```

### 5. Update Sub-Project docs/README.md

```markdown
# Documentation

**Most documentation has moved to the wrapper project.**

See: `../dnd-rulebook-project/docs/frontend/` (or `backend/`)

## What's Here

- `PROJECT-STATUS.md` - Project metrics and status
- `LATEST-HANDOVER.md` - Symlink to latest session handover

## Writing New Docs

All new plans, handovers, proposals go to the wrapper repo:
```bash
../dnd-rulebook-project/docs/frontend/plans/
../dnd-rulebook-project/docs/frontend/handovers/
```
```

## Commit Strategy

1. **Wrapper commit 1:** `docs: add frontend documentation from migration`
2. **Wrapper commit 2:** `docs: add backend documentation from migration`
3. **Wrapper commit 3:** `docs: add README explaining structure`
4. **Frontend commit:** `docs: migrate to wrapper, update CLAUDE.md`
5. **Backend commit:** `docs: migrate to wrapper, update CLAUDE.md`

## Rollback Plan

If issues arise, the wrapper repo can be reset and files restored from git history in the sub-projects (they'll still be in git history even after deletion).

## Implementation Checklist

- [ ] Run migration script (Phase 1-4)
- [ ] Verify file counts match
- [ ] Run cleanup commands (Phase 5)
- [ ] Update LATEST-HANDOVER.md symlinks
- [ ] Create wrapper docs/README.md
- [ ] Update frontend CLAUDE.md
- [ ] Update backend CLAUDE.md
- [ ] Update frontend docs/README.md
- [ ] Update backend docs/README.md
- [ ] Commit wrapper repo
- [ ] Commit frontend repo
- [ ] Commit backend repo
- [ ] Push all repos
