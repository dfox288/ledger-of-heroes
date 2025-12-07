# Parallel Agent Development

This guide explains how to run multiple Claude Code agents simultaneously, each working on different GitHub issues in isolated environments.

## Overview

The setup uses **Git worktrees** to create separate working directories that share Git history, combined with **Docker Compose overrides** to assign unique ports to each environment.

```
~/Development/dnd/
├── importer/                    # Shared backend (one instance serves all agents)
├── frontend/                    # Main working directory (port 4000)
├── frontend-agent-1/            # Agent 1 worktree (port 4001)
├── frontend-agent-2/            # Agent 2 worktree (port 4002)
└── frontend-agent-3/            # Agent 3 worktree (port 4003)
```

## Quick Start

### 1. Create a Worktree for an Agent

```bash
cd ~/Development/dnd/frontend
./scripts/create-agent-worktree.sh 1 feature/issue-130-race-subrace
```

This creates:
- `../frontend-agent-1/` — isolated working directory
- Unique ports: Nuxt=4001, HMR=34679, Storybook=6007, Nginx=8082
- Convenience scripts: `start-env.sh`, `stop-env.sh`, `run.sh`

### 2. Start the Environment

```bash
cd ~/Development/dnd/frontend-agent-1
./start-env.sh
./run.sh install    # First time only
./run.sh dev
```

### 3. Point Claude Code at the Worktree

When starting a new Claude Code session for the agent:

```bash
cd ~/Development/dnd/frontend-agent-1
claude
```

The agent will work in this isolated directory with its own Docker containers.

### 4. Clean Up When Done

```bash
cd ~/Development/dnd/frontend
./scripts/remove-agent-worktree.sh 1
```

## Port Assignments

| Instance | Nuxt | HMR | Storybook | Nginx |
|----------|------|-----|-----------|-------|
| Main (0) | 4000 | 34678 | 6006 | 8081 |
| Agent 1 | 4001 | 34679 | 6007 | 8082 |
| Agent 2 | 4002 | 34680 | 6008 | 8083 |
| Agent 3 | 4003 | 34681 | 6009 | 8084 |

## Available Scripts

| Script | Purpose |
|--------|---------|
| `scripts/create-agent-worktree.sh <N> <branch>` | Create new agent environment |
| `scripts/remove-agent-worktree.sh <N>` | Remove agent environment |
| `scripts/list-agent-worktrees.sh` | Show all active agent environments |

## Workflow Example

Working on 3 issues simultaneously:

```bash
# Terminal 1: Create environment for issue #130
./scripts/create-agent-worktree.sh 1 feature/issue-130-race-subrace
cd ../frontend-agent-1 && ./start-env.sh && ./run.sh install && ./run.sh dev

# Terminal 2: Create environment for issue #131
./scripts/create-agent-worktree.sh 2 feature/issue-131-languages
cd ../frontend-agent-2 && ./start-env.sh && ./run.sh install && ./run.sh dev

# Terminal 3: Create environment for issue #132
./scripts/create-agent-worktree.sh 3 feature/issue-132-sourcebooks
cd ../frontend-agent-3 && ./start-env.sh && ./run.sh install && ./run.sh dev
```

Then open separate Claude Code sessions in each directory.

## How It Works

### Git Worktrees

Git worktrees allow multiple working directories from a single repository. Each worktree:
- Has its own checked-out branch
- Shares the same `.git` history with the main repo
- Can have independent `node_modules`
- Commits and branches are visible from any worktree

### Docker Compose Overrides

Each worktree gets a `docker-compose.override.yml` that:
- Changes container names to avoid conflicts
- Remaps ports to unique values
- Is automatically loaded via `docker compose -f ... -f ...`

### Shared Backend

The Laravel backend (`importer/`) runs as a single instance that all frontend agents share. This is safe because:
- The API is stateless
- Each agent reads the same data
- Database changes (if any) are coordinated via Git

## Troubleshooting

### Port Already in Use

```bash
# Find what's using a port
lsof -i :4001

# List all agent environments and their status
./scripts/list-agent-worktrees.sh
```

### Container Name Conflict

If you see "container name already in use":
```bash
# Force remove the conflicting container
docker rm -f dnd-frontend-nuxt-1
```

### Worktree Locked

If git says the worktree is locked:
```bash
git worktree unlock ../frontend-agent-1
./scripts/remove-agent-worktree.sh 1
```

### Check Agent Status

```bash
./scripts/list-agent-worktrees.sh
```

Shows all agents, their branches, running status, and URLs.

## Best Practices

1. **One issue per agent** — Keep each worktree focused on a single issue/feature
2. **Commit frequently** — Changes in one worktree don't affect others until pushed
3. **Clean up finished work** — Remove worktrees after PRs are merged
4. **Use the list script** — Check `list-agent-worktrees.sh` before creating new ones
5. **Backend stays shared** — Don't create multiple backend instances unless necessary
