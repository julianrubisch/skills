---
name: jr-rails-new
description: >-
  Scaffold a new Rails app with preferred stack: PostgreSQL, Minitest, Solid
  Queue, Pundit, optional Phlex. Interactive interview, then rails new + config.
---

# Rails App Scaffolder

Interactive skill that interviews you for preferences, runs `rails new` with
the right flags, and performs post-scaffold configuration.

## Workflow

### Step 0: Preflight

Quick check that the machine can actually run `rails new`. Skip the install
prompts if everything is present.

```bash
command -v ruby   >/dev/null 2>&1 && ruby --version
command -v node   >/dev/null 2>&1 && node --version
command -v bundle >/dev/null 2>&1 && bundle --version
command -v mise   >/dev/null 2>&1 && mise --version
```

Required:

- Ruby **4.0 or newer** (parse `ruby --version`; reject 3.x and older).
- Node **20 or newer**.
- `bundle` available (ships with Ruby).
- `mise` (or `rbenv` / `asdf`) so the Ruby/Node we just probed survives a
  shell restart and matches `.tool-versions` later.

If anything's missing or out of date, surface the gap and offer to hand off:

```
This Mac isn't quite ready for `rails new` yet:
  ✗ Ruby is <version> (need 4.0+)
  ✗ Node not found
  ✓ mise installed

I can run /jr-rails-bootstrap to fix that - it installs the missing pieces
in the right order, then comes back here. Or you can install them manually
and re-run me.

Want me to launch /jr-rails-bootstrap?
```

Use `AskUserQuestion` with options `Yes, run bootstrap`, `No, I'll install manually`,
`Continue anyway (rails new may fail)`. If they pick bootstrap, invoke
`/jr-rails-bootstrap` and stop - bootstrap will return them here via Phase E.

If everything passes, print one line per tool with version and continue.

### Step 1: Interview

Ask the user each question. Show the default in brackets. Accept the default
if the user presses enter or says "default" / "yes" / "y".

| # | Question | Options | Default |
|---|----------|---------|---------|
| 1 | App name | free text | (required) |
| 2 | Database | `postgresql` / `mysql2` / `sqlite3` | `postgresql` |
| 3 | Frontend bundling | `importmap` / `esbuild` / `vite_rails` | `importmap` |
| 4 | CSS | `tailwind` / `sass` / `none` | `tailwind` |
| 5 | View layer | `erb` / `phlex` | `erb` |
| 6 | Dev container | yes / no | yes |
| 7 | Authentication | `rails` (built-in, 8+) / `devise` / none | `rails` |
| 8 | Authorization | `pundit` / none | `pundit` |
| 9 | Background jobs | `solid_queue` / `sidekiq` | `solid_queue` |
| 10 | Git worktree workflow | yes / no | no |

**Testing is always Minitest** — non-negotiable for jr-rails skills.

### Step 2: Generate

Build and run the `rails new` command:

```bash
rails new APP_NAME \
  --database=DATABASE \
  --css=CSS \
  --skip-test=false \
  --devcontainer  # if dev container = yes
```

**Bundling flags:**
- `importmap` → no extra flag (default)
- `esbuild` → `--javascript=esbuild`
- `vite_rails` → no flag; add `vite_rails` gem post-scaffold

**CSS flags:**
- `tailwind` → `--css=tailwind`
- `sass` → `--css=sass`
- `none` → `--skip-css`

### Step 3: Post-scaffold Configuration

Run these in order, skipping any that don't apply:

#### 3a. Vite (if selected)

```bash
cd APP_NAME
bundle add vite_rails
bundle exec vite install
```

Remove `importmap-rails` from Gemfile if present.

#### 3b. Phlex (if selected)

```bash
bundle add phlex-rails
```

Then create the base classes from [reference/coding-phlex.md](reference/coding-phlex.md):
- `app/components/base.rb` — `Components::Base < Phlex::HTML`
- `app/views/base.rb` — `Views::Base < Components::Base`
- `app/components/layout.rb` — `Components::Layout` with `Phlex::Rails::Layout`

Set the layout in `ApplicationController`:

```ruby
layout -> { Components::Layout }
```

Remove the ERB layout file (`app/views/layouts/application.html.erb`).

Install the custom scaffold generator from
[reference/coding-phlex.md § Scaffolding](reference/coding-phlex.md) so future
`rails g scaffold` produces Phlex views.

#### 3c. Authentication

**Rails built-in (8+):**
```bash
rails g authentication
```

**Devise:**
```bash
bundle add devise
rails g devise:install
rails g devise User
```

#### 3d. Pundit (if selected)

```bash
bundle add pundit
rails g pundit:install
```

Add to `ApplicationController`:

```ruby
class ApplicationController < ActionController::Base
  include Pundit::Authorization
  after_action :verify_authorized, except: :index
  after_action :verify_policy_scoped, only: :index
end
```

#### 3e. Sidekiq (if selected instead of Solid Queue)

```bash
bundle add sidekiq
```

Configure in `config/application.rb`:

```ruby
config.active_job.queue_adapter = :sidekiq
```

#### 3f. CLAUDE.md

Create `CLAUDE.md` in the app root with project conventions. The **How to
start** section is load-bearing: `/jr-rails-bootstrap` and future agents read
it to know how to launch the dev server. Derive its command from the actual
stack choices (Q#3 bundling, Q#6 devcontainer, Q#10 worktree).

```markdown
# Project Conventions

## Stack
- Ruby on Rails [version]
- [Database]
- [CSS framework]
- [View layer]
- Minitest with fixtures

## How to start

[Pick ONE block based on the stack choices made in the interview:]

### Default (no devcontainer, no worktree)
\`\`\`bash
bin/dev
\`\`\`
App: http://localhost:3000

### Vite (Q#3 = vite_rails) without devcontainer
\`\`\`bash
bin/dev   # foreman runs Rails + Vite together via Procfile.dev
\`\`\`
App: http://localhost:3000
Vite dev server: http://localhost:5173

### Devcontainer (Q#6 = yes)
\`\`\`bash
devcontainer up --workspace-folder .
devcontainer exec --workspace-folder . bin/dev
\`\`\`
App: http://localhost:3000

### Worktree workflow (Q#10 = yes)
Use the wt-managed binstub - port and database are derived per branch.
\`\`\`bash
wt start             # equivalent to: bin/agent-server
\`\`\`
App URL: see \`wt list\` for the branch's port (e.g. http://localhost:30421).

## Testing
- Run tests: `bin/rails test`
- Run system tests: `bin/rails test:system`

## Style
- Follow 37signals/classic Rails conventions
- Rich domain models, CRUD controllers, concerns
- No service objects - use domain models in app/models/
- Minitest with fixtures (no RSpec, no factory_bot)
- Database constraints over model validations for hard guarantees

## Skills
- `/jr-rails-classic` - coding style guide
[- `/jr-rails-phlex` - Phlex components (if Phlex selected)]
- `/jr-rails-review` - code review
```

Pick exactly **one** "How to start" block based on the interview answers and
write that block (not the multi-choice template) into the project CLAUDE.md.
The skill should resolve the right command at scaffold time, not punt the
choice to a future reader.

#### 3g. Worktree config (if selected)

Set up the agentic worktree workflow using [worktrunk](https://worktrunk.dev).
The exact binstubs and `database.yml` depend on the database choice (Q#2) and
whether dev containers are enabled (Q#6).

**1. Create `.config/wt.toml`** (same for all strategies):

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

**2. Create the binstubs** from
[reference/agentic-worktrees.md](reference/agentic-worktrees.md), choosing the
variant that matches the stack:

| Database | Dev container | Strategy | Reference section |
|----------|--------------|----------|-------------------|
| `sqlite3` | any | SQLite | § Binstubs — SQLite |
| `postgresql` / `mysql2` | no | Port-based | § Binstubs — PostgreSQL (port-based) |
| `postgresql` / `mysql2` | yes (per-worktree) | Per-worktree devcontainer | § Binstubs — PostgreSQL + Docker Compose (Per-Worktree Devcontainer) |
| `postgresql` / `mysql2` | yes (shared PG) | Shared PG | § Binstubs — PostgreSQL + Docker Compose (Shared PG) |

The **per-worktree devcontainer** strategy uses `devcontainer up` /
`devcontainer exec` (not raw `docker compose`) so that features, `containerEnv`,
and `postCreateCommand` from `devcontainer.json` are applied. The
`postCreateCommand` should be `bin/setup --skip-server`.

The **shared PG** strategy is simpler: one PostgreSQL container shared across
all worktrees, Rails runs on the host. Prefer this when the host already has
Ruby (via mise/rbenv/asdf) and you want worktrees to run tests independently.

**3. Modify `config/database.yml`** to use workspace-aware names:

- **SQLite:** prefix each database path with `AGENT_WORKSPACE` (defaults to
  `development`). See reference § `config/database.yml` (SQLite).
- **PostgreSQL:** suffix database name with `AGENT_WORKSPACE`. See reference
  § `config/database.yml` (PostgreSQL).

**4. If dev container = yes**, create `.devcontainer/agent.json` extending the
main devcontainer with `AGENT_*` env vars forwarded into the container.

**5. Update `CLAUDE.md`** with worktree workflow docs (see reference § CLAUDE.md).

**6. Update `README.md`** with the workflow (see reference § Project README).

**7. Create `.claude/settings.json`**:

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

See [reference/agentic-worktrees.md](reference/agentic-worktrees.md) for full
templates, port allocation scheme, and configuration details.

#### 3h. Verify & Commit

```bash
bin/setup
bin/rails test  # should pass with zero tests
```

Create initial commit:

```bash
git add -A
git commit -m "Initial Rails app with [stack summary]"
```

### Step 4: Summary

Print a summary of what was configured. **Derive the "Next steps" command
from the same logic as the CLAUDE.md "How to start" block** - they must
match.

```
✅ Rails app created: APP_NAME
   Database:       postgresql
   CSS:            tailwind
   View layer:     erb
   Authentication: rails (built-in)
   Authorization:  pundit
   Background:     solid_queue
   Dev container:  yes
   Worktree:       no
   Testing:        minitest (always)

Next steps:
  cd APP_NAME
  <START_COMMAND>
```

Where `<START_COMMAND>` matches the chosen stack:

| Stack                          | Command                                             |
|--------------------------------|-----------------------------------------------------|
| Default                        | `bin/dev`                                           |
| Vite (no devcontainer)         | `bin/dev` (foreman runs Rails + Vite)               |
| Devcontainer                   | `devcontainer up --workspace-folder . && devcontainer exec --workspace-folder . bin/dev` |
| Worktree workflow              | `wt start` (per-branch port; see `wt list`)         |

Then add the app URL on the next line:

```
App:  http://localhost:3000   # or the worktree-specific port
```

## Reference

- **Classic coding style**: [reference/coding-classic.md](reference/coding-classic.md)
- **Phlex coding style**: [reference/coding-phlex.md](reference/coding-phlex.md)
- **DevOps & deployment**: [reference/devops.md](reference/devops.md)
- **Gem recommendations**: [reference/toolbelt.md](reference/toolbelt.md)
