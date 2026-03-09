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

Create `CLAUDE.md` in the app root with project conventions:

```markdown
# Project Conventions

## Stack
- Ruby on Rails [version]
- [Database]
- [CSS framework]
- [View layer]
- Minitest with fixtures

## Testing
- Run tests: `bin/rails test`
- Run system tests: `bin/rails test:system`

## Style
- Follow 37signals/classic Rails conventions
- Rich domain models, CRUD controllers, concerns
- No service objects — use domain models in app/models/
- Minitest with fixtures (no RSpec, no factory_bot)
- Database constraints over model validations for hard guarantees

## Skills
- `/jr-rails-classic` — coding style guide
[- `/jr-rails-phlex` — Phlex components (if Phlex selected)]
- `/jr-rails-review` — code review
```

#### 3g. Worktree config (if selected)

Set up the agentic worktree workflow using [worktrunk](https://worktrunk.dev).
The exact setup depends on whether dev containers are also enabled (Q#6).

**If dev container = yes AND worktree = yes (full isolation):**

1. Create `.config/wt.toml`:

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

2. Create the three binstubs from
   [reference/agentic-worktrees.md](reference/agentic-worktrees.md):
   - `bin/agent-setup` — port computation, symlinks, `docker compose up -d`,
     `bin/setup`, `db:create db:schema:load`
   - `bin/agent-server` — export PORT/DATABASE_URL, run `bin/dev`
   - `bin/agent-archive` — `db:drop`, `docker compose down -v`, cleanup

3. Create `.devcontainer/agent.json` extending the main devcontainer with
   `AGENT_*` env vars forwarded into the container.

4. Modify `config/database.yml` to use workspace-aware database names:

```yaml
development:
  primary: &primary
    adapter: postgresql
    encoding: unicode
    pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
    database: <%= "APP_NAME_#{ENV.fetch('AGENT_WORKSPACE', 'development')}" %>
    host: localhost
    port: <%= ENV.fetch("AGENT_DB_PORT", 5432) %>
    username: postgres
    password: postgres
```

5. Update `CLAUDE.md` with worktree workflow docs.

**If dev container = no AND worktree = yes (port-based isolation only):**

1. Create `.config/wt.toml` (same as above).

2. Create the three binstubs (same as above, but skip `docker compose` calls —
   services run on the host with port offsets only).

3. Modify `config/database.yml` (same as above).

4. Update `CLAUDE.md` with worktree workflow docs.

**In both cases**, create `.claude/settings.json`:

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

Add a section to the project README documenting the agentic worktree workflow
(see [reference/agentic-worktrees.md § Project README](reference/agentic-worktrees.md)
for the template).

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

Print a summary of what was configured:

```
✅ Rails app created: APP_NAME
   Database:       postgresql
   CSS:            tailwind
   View layer:     erb
   Authentication: rails (built-in)
   Authorization:  pundit
   Background:     solid_queue
   Dev container:  yes
   Testing:        minitest (always)

Next steps:
  cd APP_NAME
  bin/dev
```

## Reference

- **Classic coding style**: [reference/coding-classic.md](reference/coding-classic.md)
- **Phlex coding style**: [reference/coding-phlex.md](reference/coding-phlex.md)
- **DevOps & deployment**: [reference/devops.md](reference/devops.md)
- **Gem recommendations**: [reference/toolbelt.md](reference/toolbelt.md)
