# Agentic Worktree Workflow

Isolate concurrent AI agent sessions so each gets its own database, services,
and port range — without stepping on other agents or your main dev environment.

## Overview

Each agent gets a **git worktree** (managed by [worktrunk](https://worktrunk.dev))
with isolated infrastructure: its own database, Redis, Selenium, and port range.
Two isolation strategies:

1. **Docker Compose per workspace** (recommended) — each worktree spins up its
   own containers with workspace-prefixed names and port offsets
2. **Port-based isolation only** — lightweight fallback using workspace-suffixed
   database names and computed port offsets (no extra containers)

### Prerequisites

- [worktrunk](https://worktrunk.dev) (`cargo install wt-cli` or `brew install worktrunk`)
- Docker + Docker Compose (for strategy 1)
- `devcontainer-cli` (optional, for devcontainer variant)

## Worktrunk Configuration

Create `.config/wt.toml` in the project root:

```toml
[worktree]
path = "../.worktrees/{branch}"

[hooks]
post-create = "bin/agent-setup"
pre-remove = "bin/agent-archive"

[hooks.env]
AGENT_ROOT_PATH = "{root}"
AGENT_WORKSPACE = "{branch}"
```

### Key worktrunk config options

| Option | Description |
|--------|-------------|
| `worktree.path` | Template for worktree location. `{branch}` expands to branch name. Placing outside the repo (with `../`) avoids nested `.git` issues. |
| `hooks.post-create` | Script run after `wt switch` creates a new worktree. Use for DB setup, container start. |
| `hooks.pre-remove` | Script run before `wt remove` tears down a worktree. Use for DB drop, container cleanup. |
| `hooks.pre-merge` | Script run before `wt merge`. Use for pre-merge checks or test runs. |
| `hooks.env` | Environment variables passed to all hooks. `{root}` = main worktree path, `{branch}` = worktree branch name. |
| `shared-cache` | Directories to share across worktrees (e.g. `node_modules`, `vendor/bundle`) via symlinks. Avoids redundant installs. |

### Agentic workflow with worktrunk

```bash
# Create a new workspace for an agent
wt switch feature-auth

# worktrunk:
# 1. Creates worktree at ../.worktrees/feature-auth
# 2. Runs bin/agent-setup with AGENT_WORKSPACE=feature-auth

# Agent works in the worktree...

# Merge back to main
wt merge feature-auth

# Tear down when done
wt remove feature-auth
# Runs bin/agent-archive before removing
```

## Port Allocation Scheme

Each workspace gets a range of **10 consecutive ports** derived deterministically
from the workspace name using CRC32:

```
AGENT_PORT = (CRC32(workspace_name) % 100) * 10 + 3000
```

| Offset | Service | Example (base=3010) |
|--------|---------|---------------------|
| +0 | Rails / Puma | 3010 |
| +1 | PostgreSQL | 3011 |
| +2 | Redis | 3012 |
| +3 | Selenium Chrome | 3013 |
| +4 | Vite / esbuild dev server | 3014 |
| +5..+9 | Reserved for app-specific services | 3015–3019 |

The default development environment (no workspace) uses standard ports (3000, 5432, 6379, etc.).

## Docker Compose Per-Workspace Isolation

Add a `compose.yaml` to the project root (or `.devcontainer/compose.yaml`).
Services use `AGENT_WORKSPACE` for container naming and `AGENT_PORT`-derived
variables for port mapping:

```yaml
services:
  db:
    image: postgres:17
    container_name: "${AGENT_WORKSPACE:-dev}_db"
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "${AGENT_DB_PORT:-5432}:5432"
    volumes:
      - "db_data_${AGENT_WORKSPACE:-dev}:/var/lib/postgresql/data"

  redis:
    image: redis:7
    container_name: "${AGENT_WORKSPACE:-dev}_redis"
    ports:
      - "${AGENT_REDIS_PORT:-6379}:6379"

  chrome:
    image: selenium/standalone-chromium
    container_name: "${AGENT_WORKSPACE:-dev}_chrome"
    ports:
      - "${AGENT_CHROME_PORT:-4444}:4444"

volumes:
  db_data_${AGENT_WORKSPACE:-dev}:
```

Each workspace gets its own named volume, so databases are fully isolated.

## Binstubs

### `bin/agent-setup` (worktrunk post-create hook)

```bash
#!/bin/bash
set -euo pipefail

# Compute port range from workspace name
WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"
ROOT="${AGENT_ROOT_PATH:?AGENT_ROOT_PATH must be set}"

# CRC32-based port allocation (deterministic, no collisions for typical branch counts)
CRC=$(printf '%s' "$WORKSPACE" | cksum | cut -d' ' -f1)
export AGENT_PORT=$(( (CRC % 100) * 10 + 3000 ))
export AGENT_DB_PORT=$(( AGENT_PORT + 1 ))
export AGENT_REDIS_PORT=$(( AGENT_PORT + 2 ))
export AGENT_CHROME_PORT=$(( AGENT_PORT + 3 ))
export AGENT_VITE_PORT=$(( AGENT_PORT + 4 ))

echo "==> Agent workspace: $WORKSPACE"
echo "==> Port range: $AGENT_PORT - $(( AGENT_PORT + 9 ))"

# Symlink shared resources from main worktree
ln -sf "$ROOT/.env" .env 2>/dev/null || true
ln -sf "$ROOT/.bundle" .bundle 2>/dev/null || true
ln -sf "$ROOT/storage" storage 2>/dev/null || true

# Copy credentials (can't symlink — Rails reads relative to config/)
mkdir -p config/credentials
cp "$ROOT"/config/credentials/*.key config/credentials/ 2>/dev/null || true

# Start workspace containers
docker compose up -d

# Install dependencies and set up database
bin/setup
bin/rails db:create db:schema:load

echo "==> Workspace $WORKSPACE ready on port $AGENT_PORT"
```

### `bin/agent-server` (run the app in this workspace)

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:-$(git branch --show-current)}"

# Compute port range (same algorithm as agent-setup)
CRC=$(printf '%s' "$WORKSPACE" | cksum | cut -d' ' -f1)
export AGENT_PORT=$(( (CRC % 100) * 10 + 3000 ))
export PORT=$AGENT_PORT
export AGENT_DB_PORT=$(( AGENT_PORT + 1 ))
export VITE_RUBY_PORT=$(( AGENT_PORT + 4 ))
export DATABASE_URL="postgres://postgres:postgres@localhost:$AGENT_DB_PORT/myapp_${WORKSPACE}"

echo "==> Starting $WORKSPACE on port $PORT"
exec bin/dev
```

### `bin/agent-archive` (worktrunk pre-remove hook)

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"

echo "==> Archiving workspace: $WORKSPACE"

# Drop the workspace database
bin/rails db:drop 2>/dev/null || true

# Stop containers and remove volumes
docker compose down -v 2>/dev/null || true

# Remove symlinks
rm -f .env .bundle storage 2>/dev/null || true

echo "==> Workspace $WORKSPACE archived"
```

## Database Isolation

Modify `config/database.yml` to use workspace-specific database names and ports:

```yaml
development:
  primary: &primary
    adapter: postgresql
    encoding: unicode
    pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
    database: <%= "myapp_#{ENV.fetch('AGENT_WORKSPACE', 'development')}" %>
    host: localhost
    port: <%= ENV.fetch("AGENT_DB_PORT", 5432) %>
    username: postgres
    password: postgres
```

Each workspace gets a separate database (`myapp_feature-auth`, `myapp_fix-login`, etc.).
The default development environment uses `myapp_development` on port 5432.

## Rails Configuration

### Puma

Bind to the workspace port:

```ruby
# config/puma.rb
port ENV.fetch("PORT", 3000)
bind "tcp://0.0.0.0:#{ENV.fetch('PORT', 3000)}"
```

### Action Mailer

Use the workspace port for URL generation:

```ruby
# config/environments/development.rb
config.action_mailer.default_url_options = {
  host: "localhost",
  port: ENV.fetch("PORT", 3000)
}
```

### Selenium / System Tests

Point Capybara at the workspace Chrome container:

```ruby
# test/application_system_test_case.rb
driven_by :selenium, using: :headless_chromium, screen_size: [1400, 900],
  options: {
    browser: :remote,
    url: "http://localhost:#{ENV.fetch('AGENT_CHROME_PORT', 4444)}/wd/hub"
  }
```

## Active Storage

Symlink `storage/` to the main worktree so file uploads are shared:

```bash
ln -sf "$AGENT_ROOT_PATH/storage" storage
```

This is handled automatically in `bin/agent-setup`.

## Claude Code Integration

### `.claude/settings.json`

Grant permissions for worktree operations:

```json
{
  "permissions": {
    "allow": [
      "Bash(git worktree *)",
      "Bash(git branch *)",
      "Bash(wt *)",
      "Bash(bin/agent-*)",
      "Bash(docker compose *)"
    ]
  }
}
```

### CLAUDE.md

Add a section documenting the worktree workflow:

```markdown
## Worktree Workflow

This project uses worktrunk for agent workspace isolation.

- Create a workspace: `wt switch <branch-name>`
- Run the app: `bin/agent-server`
- Tear down: `wt remove <branch-name>`

Each workspace gets its own database and port range. See
`reference/agentic-worktrees.md` for details.
```

## Project README

Always document the agentic worktree workflow in the project's README so
developers and agents can discover it:

```markdown
## Development

### Agentic Worktree Workflow (optional)

This project supports isolated agent workspaces via
[worktrunk](https://worktrunk.dev). Each workspace gets its own database,
port range, and Docker containers.

```bash
# Create a new workspace
wt switch feature-auth

# Run the app in this workspace
bin/agent-server

# Tear down when done
wt remove feature-auth
```

See `bin/agent-setup`, `bin/agent-server`, `bin/agent-archive` for details.
```

## Devcontainer Variant

For full container isolation, extend the main devcontainer config with an
agent-specific variant. Create `.devcontainer/agent.json`:

```json
{
  "name": "Agent Workspace",
  "dockerComposeFile": ["compose.yaml"],
  "service": "app",
  "workspaceFolder": "/workspace",
  "containerEnv": {
    "AGENT_WORKSPACE": "${localEnv:AGENT_WORKSPACE}",
    "AGENT_PORT": "${localEnv:AGENT_PORT}",
    "AGENT_DB_PORT": "${localEnv:AGENT_DB_PORT}",
    "AGENT_REDIS_PORT": "${localEnv:AGENT_REDIS_PORT}",
    "AGENT_CHROME_PORT": "${localEnv:AGENT_CHROME_PORT}"
  },
  "postCreateCommand": "bin/agent-setup",
  "features": {
    "ghcr.io/devcontainers/features/ruby:1": {},
    "ghcr.io/devcontainers/features/node:1": {}
  }
}
```

This forwards all `AGENT_*` env vars into the container, so port allocation
and database isolation work identically inside or outside containers.
