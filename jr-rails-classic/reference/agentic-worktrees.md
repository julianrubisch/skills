# Agentic Worktree Workflow

Isolate concurrent AI agent sessions so each gets its own database, services,
and port range — without stepping on other agents or your main dev environment.

## Overview

Each agent gets a **git worktree** (managed by [worktrunk](https://worktrunk.dev))
with isolated infrastructure: its own database and port range. Four isolation
strategies depending on your stack:

1. **SQLite** — workspace-prefixed SQLite files in `storage/`, no containers
2. **PostgreSQL (port-based)** — workspace-suffixed database names with port
   offsets, services on the host
3. **PostgreSQL + Docker Compose (per-worktree devcontainer)** — each worktree
   spins up its own devcontainer stack with parameterized compose, random ports,
   and `bin/agent-exec` for running commands inside the container
4. **PostgreSQL + Docker Compose (shared PG)** — one PostgreSQL container shared
   across all worktrees, isolation via database naming. Rails runs on the host.

### Prerequisites

- [worktrunk](https://worktrunk.dev) (`brew install worktrunk`)
- Docker + Docker Compose (for strategy 3 only)

## Worktrunk Configuration

Create `.config/wt.toml` in the project root. Worktrunk uses Tera templates
with `{{ branch }}` variables and built-in filters.

```toml
[list]
url = "http://localhost:{{ branch | hash_port }}"

[post-create]
setup = "AGENT_WORKSPACE={{ branch | sanitize }} AGENT_ROOT_PATH={{ primary_worktree_path }} bin/agent-setup"

[pre-remove]
cleanup = "AGENT_WORKSPACE={{ branch | sanitize }} bin/agent-cleanup"

[post-remove]
kill-server = "lsof -ti :{{ branch | hash_port }} -sTCP:LISTEN | xargs kill 2>/dev/null || true"

[post-start]
server = "PORT={{ branch | hash_port }} VITE_RUBY_PORT={{ (branch ~ '-vite') | hash_port }} bin/agent-server"
```

The `[post-start]` hook runs `bin/agent-server` with PORT and VITE_RUBY_PORT
pre-computed by worktrunk's `hash_port` filter. The Vite port uses
`{{ (branch ~ '-vite') | hash_port }}` — the `~` operator concatenates strings,
so a different input produces a different deterministic port.

### Key worktrunk template variables and filters

| Variable / Filter | Description |
|-------------------|-------------|
| `{{ branch }}` | Branch name of the worktree |
| `{{ primary_worktree_path }}` | Absolute path to the main (primary) worktree |
| `{{ branch \| sanitize }}` | Branch name with `/` replaced by `-`, safe for filenames and env vars |
| `{{ branch \| hash_port }}` | Deterministic port (10000–19999) derived from branch name via CRC32 |
| `{{ (branch ~ '-vite') \| hash_port }}` | Concatenate suffix with `~`, then hash — gives a separate deterministic port |

### Hook sections

| Section | When it runs |
|---------|-------------|
| `[post-create]` | After `wt switch --create` creates a new worktree |
| `[post-start]` | When `wt start` runs in an existing worktree |
| `[pre-remove]` | Before `wt remove` tears down a worktree |
| `[post-remove]` | After `wt remove` finishes removing the worktree |
| `[pre-merge]` | Before `wt merge` merges a worktree branch |

Each section contains named commands (e.g. `setup = "..."`) that run in order.
Environment variables are passed inline, not via a separate `[hooks.env]` section.

### Agentic workflow with worktrunk

```bash
# Create a new workspace (branches from main by default)
wt switch --create feature-auth

# Create from a specific branch
wt switch --create feature-auth --base other-branch

# worktrunk:
# 1. Creates worktree
# 2. Runs bin/agent-setup with AGENT_WORKSPACE=feature-auth

# Agent works in the worktree...

# Run the app (via post-start hook, or manually)
wt start feature-auth
# or: bin/agent-server

# Merge back to main
wt merge feature-auth

# Tear down when done
wt remove feature-auth
# Runs bin/agent-cleanup, then kills the server process
```

## Binstubs — SQLite

### `bin/agent-setup`

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"
ROOT="${AGENT_ROOT_PATH:?AGENT_ROOT_PATH must be set}"

echo "==> Agent workspace: $WORKSPACE"

# Symlink shared resources from main worktree
ln -sf "$ROOT/.env" .env 2>/dev/null || true
ln -sf "$ROOT/.bundle" .bundle 2>/dev/null || true
ln -sf "$ROOT/node_modules" node_modules 2>/dev/null || true

# Copy credentials (can't symlink — Rails reads relative to config/)
mkdir -p config/credentials
cp "$ROOT"/config/credentials/*.key config/credentials/ 2>/dev/null || true
cp "$ROOT"/config/master.key config/ 2>/dev/null || true

# Install dependencies and set up database
# --skip-server prevents bin/dev from replacing this process
bin/setup --skip-server

echo "==> Workspace $WORKSPACE ready"
```

### `bin/agent-server`

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:-$(git branch --show-current)}"
export AGENT_WORKSPACE="$WORKSPACE"

# Accept PORT from env (e.g. set by worktrunk's hash_port), or fall back to 3000
export PORT="${PORT:-3000}"
export VITE_RUBY_PORT="${VITE_RUBY_PORT:-$(( PORT + 1 ))}"

echo "==> Starting $WORKSPACE on port $PORT (Vite: $VITE_RUBY_PORT)"
exec bin/dev
```

### `bin/agent-cleanup`

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"

echo "==> Cleaning up workspace: $WORKSPACE"

# Remove workspace-specific SQLite databases
rm -f storage/"${WORKSPACE}".sqlite3 2>/dev/null || true
rm -f storage/"${WORKSPACE}"-*.sqlite3 2>/dev/null || true

# Remove symlinks
rm -f .env .bundle node_modules 2>/dev/null || true

echo "==> Workspace $WORKSPACE cleaned up"
```

### `config/database.yml` (SQLite)

```yaml
development:
  primary:
    <<: *default
    database: storage/<%= ENV.fetch('AGENT_WORKSPACE', 'development') %>.sqlite3
  queue:
    <<: *queue
    database: storage/<%= ENV.fetch('AGENT_WORKSPACE', 'development') %>-queue.sqlite3
  cache:
    <<: *cache
    database: storage/<%= ENV.fetch('AGENT_WORKSPACE', 'development') %>-cache.sqlite3
  cable:
    <<: *cable
    database: storage/<%= ENV.fetch('AGENT_WORKSPACE', 'development') %>-cable.sqlite3
  errors:
    <<: *errors
    database: storage/<%= ENV.fetch('AGENT_WORKSPACE', 'development') %>-errors.sqlite3
```

When `AGENT_WORKSPACE` is unset, defaults to `development` — no change for
normal dev. Each workspace gets isolated files (`storage/feature-auth.sqlite3`,
`storage/feature-auth-queue.sqlite3`, etc.).

## Binstubs — PostgreSQL (port-based, no Docker)

### `bin/agent-setup`

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"
ROOT="${AGENT_ROOT_PATH:?AGENT_ROOT_PATH must be set}"

echo "==> Agent workspace: $WORKSPACE"

# Symlink shared resources from main worktree
ln -sf "$ROOT/.env" .env 2>/dev/null || true
ln -sf "$ROOT/.bundle" .bundle 2>/dev/null || true
ln -sf "$ROOT/node_modules" node_modules 2>/dev/null || true
ln -sf "$ROOT/storage" storage 2>/dev/null || true

# Copy credentials (can't symlink — Rails reads relative to config/)
mkdir -p config/credentials
cp "$ROOT"/config/credentials/*.key config/credentials/ 2>/dev/null || true
cp "$ROOT"/config/master.key config/ 2>/dev/null || true

# Install dependencies and set up database
bin/setup --skip-server

echo "==> Workspace $WORKSPACE ready"
```

### `bin/agent-server`

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:-$(git branch --show-current)}"
export AGENT_WORKSPACE="$WORKSPACE"

# Accept PORT from env (e.g. set by worktrunk's hash_port), or fall back to 3000
export PORT="${PORT:-3000}"
export VITE_RUBY_PORT="${VITE_RUBY_PORT:-$(( PORT + 1 ))}"
export DATABASE_URL="postgres://localhost/myapp_${WORKSPACE}"

echo "==> Starting $WORKSPACE on port $PORT (Vite: $VITE_RUBY_PORT)"
exec bin/dev
```

### `bin/agent-cleanup`

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"

echo "==> Cleaning up workspace: $WORKSPACE"

# Drop the workspace database
bin/rails db:drop 2>/dev/null || true

# Remove symlinks
rm -f .env .bundle node_modules storage 2>/dev/null || true

echo "==> Workspace $WORKSPACE cleaned up"
```

### `config/database.yml` (PostgreSQL)

```yaml
development:
  primary: &primary
    adapter: postgresql
    encoding: unicode
    pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
    database: <%= "myapp_#{ENV.fetch('AGENT_WORKSPACE', 'development')}" %>
    host: localhost
    username: postgres
    password: postgres
```

## Binstubs — PostgreSQL + Docker Compose (Per-Worktree Devcontainer)

Each worktree gets its own devcontainer stack with isolated containers, volumes,
and ports. The `.devcontainer/compose.yaml` is parameterized with env vars so
the same file works for both the primary worktree and agent worktrees.

### `.devcontainer/compose.yaml` changes

Parameterize the compose file so each worktree gets its own project name,
volume mount, and port bindings:

```yaml
name: "${COMPOSE_PROJECT_NAME:-myapp}"
# Each worktree uses "myapp_<branch>" so containers don't collide.

services:
  rails-app:
    volumes:
      - ${WORKTREE_PATH:-..}:/workspaces/myapp:cached
      # Uses WORKTREE_PATH (absolute path to worktree) or defaults to ".."
      # (primary worktree). Always mounts at /workspaces/myapp for consistency.

    ports:
      - "${APP_PORT:-3000}:3000"
      - "${VITE_PORT:-3036}:3036"
      # Each worktree uses random ports (APP_PORT=0) to avoid collisions.
      # Primary keeps defaults 3000/3036.

  postgres:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD: postgres
    volumes:
      - db_data:/var/lib/postgresql/data

volumes:
  db_data:
```

### `bin/agent-setup`

Uses `devcontainer up` to spin up the container. This ensures devcontainer
features (Node.js, yarn, etc.) and `containerEnv` (like `DB_HOST`) from
`devcontainer.json` are applied — raw `docker compose up` would skip those.

The `postCreateCommand` in `devcontainer.json` should run
`bin/setup --skip-server`, which handles `bundle install`, `yarn install`,
and `db:prepare` when the container is first created.

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"
ROOT="${AGENT_ROOT_PATH:?AGENT_ROOT_PATH must be set}"
WORKTREE_PATH="$(pwd)"

echo "==> Agent workspace: $WORKSPACE"

# Symlink shared resources from main worktree
# Don't symlink node_modules — postCreateCommand installs inside the container
ln -sf "$ROOT/.env" .env 2>/dev/null || true
ln -sf "$ROOT/.bundle" .bundle 2>/dev/null || true
ln -sf "$ROOT/storage" storage 2>/dev/null || true

# Copy credentials (can't symlink — Rails reads relative to config/)
mkdir -p config/credentials
cp "$ROOT"/config/credentials/*.key config/credentials/ 2>/dev/null || true
cp "$ROOT"/config/master.key config/ 2>/dev/null || true

# Start isolated devcontainer for this worktree
# devcontainer up applies features, containerEnv, and postCreateCommand
export COMPOSE_PROJECT_NAME="myapp_${WORKSPACE}"
export WORKTREE_PATH
export APP_PORT=0    # random available port
export VITE_PORT=0

devcontainer up --workspace-folder "$WORKTREE_PATH"

echo "==> Workspace $WORKSPACE ready"
```

### `bin/agent-cleanup`

Tears down the worktree's container stack. Auto-detects ROOT from git if
`AGENT_ROOT_PATH` isn't set:

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"
ROOT="${AGENT_ROOT_PATH:-$(git -C "$(dirname "$0")/.." worktree list --porcelain | head -1 | cut -d' ' -f2)}"

echo "==> Cleaning up workspace: $WORKSPACE"

# Tear down the worktree's devcontainer stack
export COMPOSE_PROJECT_NAME="myapp_${WORKSPACE}"
docker compose -f "$ROOT/.devcontainer/compose.yaml" down -v 2>/dev/null || true

# Remove symlinks
rm -f .env .bundle node_modules storage 2>/dev/null || true

echo "==> Workspace $WORKSPACE cleaned up"
```

### `bin/agent-exec`

Convenience script to run commands inside a worktree's container. Uses
`devcontainer exec` instead of `docker compose exec` so that `containerEnv`
from `devcontainer.json` is automatically applied:

```bash
#!/bin/bash
set -euo pipefail

# Run a command inside this worktree's devcontainer.
# Usage: bin/agent-exec bin/rails test
#        bin/agent-exec bin/ci
#        bin/agent-exec bin/rubocop -a

WORKTREE_PATH="$(pwd)"

devcontainer exec --workspace-folder "$WORKTREE_PATH" bash -ic "$*"
```

### CLAUDE.md addition

Document `bin/agent-exec` as the way to run tests/CI in worktrees:

```markdown
## Running Commands in Worktrees

Use `bin/agent-exec` to run commands inside the worktree's container:

```bash
bin/agent-exec bin/rails test
bin/agent-exec bin/ci
bin/agent-exec bin/rubocop -a
```
```

## Binstubs — PostgreSQL + Docker Compose (Shared PG)

When using devcontainers with worktrees, each worktree needs to run
`bin/rails test` independently. The default devcontainer setup mounts only the
primary worktree, so secondary worktrees can't run tests inside the container
without copying files.

The solution: expose PostgreSQL from the devcontainer on a host port. Each
worktree connects to the same PG instance but uses a workspace-specific database
name. Rails runs natively on the host (via mise/rbenv/asdf), not inside the
container.

### Prerequisites

The host must have Ruby + bundler available (via mise, rbenv, or asdf). The
devcontainer is only used for PostgreSQL (and optionally Selenium for system
tests).

### `compose.yaml`

Expose PG port to the host:

```yaml
services:
  postgres:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
```

### `config/database.yml`

The primary worktree (no `AGENT_WORKSPACE` set) gets the default DB names.
Each worktree gets a suffixed name like `app_name_test_feature-branch`.

```yaml
default: &default
  adapter: postgresql
  encoding: unicode
  host: localhost
  port: 5432
  username: postgres
  password: postgres

development:
  <<: *default
  database: <%= "app_name_development#{ENV['AGENT_WORKSPACE'] ? '_' + ENV['AGENT_WORKSPACE'] : ''}" %>

test:
  <<: *default
  database: <%= "app_name_test#{ENV['AGENT_WORKSPACE'] ? '_' + ENV['AGENT_WORKSPACE'] : ''}" %>
```

### `bin/agent-setup`

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"
ROOT="${AGENT_ROOT_PATH:?AGENT_ROOT_PATH must be set}"

echo "==> Agent workspace: $WORKSPACE"

# Symlink shared resources from main worktree
ln -sf "$ROOT/.env" .env 2>/dev/null || true
ln -sf "$ROOT/node_modules" node_modules 2>/dev/null || true

# Ensure PG is running (from primary worktree's compose)
docker compose -f "$ROOT/.devcontainer/compose.yaml" up -d postgres

# Setup database (runs on host, connects to PG via localhost)
bin/rails db:create db:migrate

echo "==> Workspace $WORKSPACE ready"
```

### `bin/agent-cleanup`

```bash
#!/bin/bash
set -euo pipefail

WORKSPACE="${AGENT_WORKSPACE:?AGENT_WORKSPACE must be set}"

echo "==> Cleaning up workspace: $WORKSPACE"
bin/rails db:drop 2>/dev/null || true
rm -f .env node_modules 2>/dev/null || true
echo "==> Workspace $WORKSPACE cleaned up"
```

### Key points

- PostgreSQL container is shared across all worktrees — only one instance runs
- Database isolation is via naming, not separate PG instances
- Rails runs on the host (requires Ruby available via mise/rbenv/asdf)
- `bin/agent-setup` ensures PG is up before creating the workspace DB
- The primary worktree can still use `devcontainer exec` for its own workflow
- `bin/ci` and `bin/rails test` run directly in each worktree, no file copying
- Subagents spawned with `isolation: "worktree"` or via `wt switch` both work,
  as long as `AGENT_WORKSPACE` is set

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

Point Capybara at the workspace Chrome container (Docker strategy only):

```ruby
# test/application_system_test_case.rb
driven_by :selenium, using: :headless_chromium, screen_size: [1400, 900],
  options: {
    browser: :remote,
    url: "http://localhost:#{ENV.fetch('AGENT_CHROME_PORT', 4444)}/wd/hub"
  }
```

## Active Storage

For PostgreSQL strategies, symlink `storage/` to the main worktree so file
uploads are shared:

```bash
ln -sf "$AGENT_ROOT_PATH/storage" storage
```

This is handled automatically in `bin/agent-setup`.

For SQLite, do **not** symlink `storage/` — the workspace-prefixed SQLite
database files live there.

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
      "Bash(docker compose *)",
      "Bash(devcontainer *)"
    ]
  }
}
```

### CLAUDE.md

Add a section documenting the worktree workflow:

```markdown
## Worktree Workflow

This project uses [worktrunk](https://worktrunk.dev) for agent workspace
isolation. Each workspace gets its own database and port range.

```bash
# Create a new workspace (branches from main by default)
wt switch --create feature-auth

# Create from a specific branch
wt switch --create feature-auth --base other-branch

# Run the app in this workspace
bin/agent-server

# Tear down when done
wt remove feature-auth
```

See `bin/agent-setup`, `bin/agent-server`, `bin/agent-cleanup` for details.
```

## Port Allocation

Worktrunk's `hash_port` filter computes a deterministic port in the
**10000–19999** range from the branch name using CRC32. The `[post-start]` hook
passes these ports to `bin/agent-server` via environment variables, so the
binstubs themselves don't need to compute ports.

For services that need separate ports (e.g. Vite), use the Tera `~` operator to
concatenate a suffix before hashing: `{{ (branch ~ '-vite') | hash_port }}`
produces a different deterministic port.

For Docker Compose, `bin/agent-setup` replicates the hash_port algorithm in
bash to set container port mappings:

| Offset | Service | Example (base=13042) |
|--------|---------|----------------------|
| +0 | Rails / Puma | 13042 |
| +1 | (reserved for Vite — uses separate hash) | — |
| +2 | PostgreSQL | 13044 |
| +3 | Redis | 13045 |
| +4 | Selenium Chrome | 13046 |
| +5..+9 | Reserved for app-specific services | 13047–13051 |

The default development environment (no `AGENT_WORKSPACE`) uses standard ports.

## Project README

Always document the agentic worktree workflow in the project's README so
developers and agents can discover it:

```markdown
## Development

### Agentic Worktree Workflow (optional)

This project supports isolated agent workspaces via
[worktrunk](https://worktrunk.dev). Each workspace gets its own database
and port range, so multiple agents can work concurrently without conflicts.

```bash
# Install worktrunk
brew install worktrunk

# Create a new workspace (branches from main by default)
wt switch --create feature-auth

# Create from a specific branch
wt switch --create feature-auth --base other-branch

# Run the app in this workspace
bin/agent-server

# Tear down when done
wt remove feature-auth
```

See `bin/agent-setup`, `bin/agent-server`, `bin/agent-cleanup` for details.
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
  "postCreateCommand": "bin/setup --skip-server",
  "features": {
    "ghcr.io/devcontainers/features/ruby:1": {},
    "ghcr.io/devcontainers/features/node:1": { "installYarn": true }
  }
}
```

This forwards all `AGENT_*` env vars into the container, so port allocation
and database isolation work identically inside or outside containers.

### Devcontainer Gotchas

**Use `devcontainer up`, not `docker compose up`.** When spinning up worktree
containers, always use `devcontainer up --workspace-folder "$WORKTREE_PATH"`
instead of raw `docker compose up -d --build`. Only `devcontainer up` applies
devcontainer features (Node.js, yarn, etc.), `containerEnv` from
`devcontainer.json` (like `DB_HOST`), and runs `postCreateCommand`. Similarly,
use `devcontainer exec --workspace-folder "$WORKTREE_PATH"` instead of
`docker compose exec` to pick up `containerEnv` automatically. Cleanup with
`docker compose down -v` is fine — there's no `devcontainer down`.

**`forwardPorts` only works in VS Code.** When using `devcontainer up` from the
CLI (e.g. for headless agent sessions), `forwardPorts` in `devcontainer.json`
is silently ignored. You must expose ports via `ports:` in `compose.yaml`
instead. This applies to Rails, Vite, PostgreSQL, Redis — anything that needs
to be reachable from the host.

**Node.js must be added explicitly.** The Rails devcontainer base image does not
include Node.js. If the project uses `vite_rails`, `esbuild`, or any npm/yarn
packages, add the Node.js feature:

```json
{
  "features": {
    "ghcr.io/devcontainers/features/node:1": {}
  }
}
```

Always include this feature by default — even projects that start without JS
dependencies often add them later. Prefer yarn as the package manager:

```json
{
  "features": {
    "ghcr.io/devcontainers/features/node:1": {
      "installYarn": true
    }
  }
}
```

Without this, `bin/dev` will fail when the Vite/esbuild process tries to start.

**Process manager must be installed.** The base image does not include
`overmind` or `foreman`, which `bin/dev` (via `Procfile.dev`) requires.
Preferred approach — install overmind in the Dockerfile (requires tmux):

```dockerfile
# Install overmind process manager (requires tmux)
USER root
RUN apt-get update && apt-get install -y --no-install-recommends tmux \
    && rm -rf /var/lib/apt/lists/* \
    && curl -Lo /tmp/overmind.gz https://github.com/DarthSim/overmind/releases/download/v2.5.1/overmind-v2.5.1-linux-arm64.gz \
    && gunzip -c /tmp/overmind.gz > /usr/local/bin/overmind \
    && chmod +x /usr/local/bin/overmind \
    && rm /tmp/overmind.gz
USER vscode
```

Alternatively, add `gem "foreman"` to the Gemfile (development group).
