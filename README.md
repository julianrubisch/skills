# jr-rails-skills

Rails coding skills for Claude Code. Opinionated, production-tested patterns
following 37signals/classic conventions.

## Installation

```bash
npx skills add julianrubisch/skills
```

Or via the plugin marketplace:

```
/plugin marketplace add julianrubisch/skills
/plugin install jr-rails-classic@julianrubisch-skills
```

## Available Skills

| Skill | Description |
|-------|-------------|
| `jr-rails-classic` | Write Rails code in 37signals/classic style — rich models, CRUD controllers, concerns, state-as-records, Minitest with fixtures |
| `jr-rails-new` | Scaffold a new Rails app with preferred stack — interactive interview, then `rails new` + full post-scaffold configuration |
| `jr-rails-phlex` | Write Phlex views and components for Rails — class hierarchy, slots, helpers, custom elements, scaffold generator |

## jr-rails-classic

Guides Claude to follow 37signals conventions when writing or modifying Rails
application code.

**Invoke:** `/jr-rails-classic`

Core workflow: generators → models → controllers → views → tests.

Key conventions enforced:
- No service objects — domain models in `app/models/`
- No custom controller actions — sub-resources for everything (REST mapping)
- Database-backed state (records with timestamps, not booleans)
- Callbacks only for derived data and async dispatch
- Solid Queue, Solid Cache, Solid Cable (no Redis)
- Minitest with fixtures (no RSpec, no factory_bot)

Includes deep reference material loaded on demand:
- 7 design patterns (form objects, query objects, strategies, etc.)
- Anti-patterns and code smells with prioritized severity
- 9 refactoring recipes with before/after examples
- Testing guide (Minitest, fixtures, per-layer focus)
- Categorized gem toolbelt
- Hotwire, background jobs, state machines, authorization, notifications,
  instrumentation, and more

## jr-rails-new

Interactive scaffolder that interviews you for preferences, runs `rails new`,
and performs full post-scaffold configuration.

**Invoke:** `/jr-rails-new`

Interview questions:

| Question | Options | Default |
|----------|---------|---------|
| App name | free text | (required) |
| Database | PostgreSQL / MySQL / SQLite | PostgreSQL |
| Frontend bundling | importmap / esbuild / vite_rails | importmap |
| CSS | Tailwind / Sass / none | Tailwind |
| View layer | ERB / Phlex | ERB |
| Dev container | yes / no | yes |
| Authentication | Rails built-in / Devise / none | Rails |
| Authorization | Pundit / none | Pundit |
| Background jobs | Solid Queue / Sidekiq | Solid Queue |
| Git worktree workflow | yes / no | no |

Testing is always **Minitest with fixtures**.

Post-scaffold steps include Phlex base classes + custom scaffold generator
(if selected), Pundit install, `CLAUDE.md` with project conventions, and
optional agentic worktree setup for multi-agent development.

## jr-rails-phlex

Guides Claude to build UI with Phlex views and components.

**Invoke:** `/jr-rails-phlex`

Covers:
- `Components::Base` / `Views::Base` class hierarchy
- Short-form component calls (`PageHeader(title: "Labels")`)
- Slots via public methods (no DSL)
- Content areas and multiple layouts
- Custom element wrappers (`register_element`)
- Controller rendering patterns
- Frontend integration: Stimulus, Turbo Frames/Streams, Pagy
- ERB partials for forms (pragmatic escape hatch)
- Fragment caching

## Reference Library

Each skill includes a `reference/` directory with detailed guides that Claude
reads on demand — not loaded all at once.

**Shared references** (cross-cutting, used by multiple skills):

| File | Content |
|------|---------|
| `shared/architecture.md` | Layered architecture (4 layers, rules, violations) |
| `shared/authorization.md` | Pundit policies |
| `shared/callbacks.md` | Callback scoring, extraction signals |
| `shared/components.md` | Phlex components deep dive |
| `shared/concerns.md` | Concern design heuristics |
| `shared/configuration.md` | Anyway Config for complex config |
| `shared/current_attributes.md` | Current usage rules |
| `shared/hotwire.md` | Turbo + Stimulus |
| `shared/instrumentation.md` | Rails.event (8.1+) |
| `shared/jobs.md` | ActiveJob + Solid Queue + Continuations |
| `shared/notifications.md` | Noticed gem |
| `shared/security.md` | Security reference |
| `shared/serializers.md` | SimpleDelegator + AMS |
| `shared/state_machines.md` | AASM |
| `shared/testing.md` | Minitest, fixtures, test pyramid |

**Refactoring recipes** (9 recipes in `reference/refactorings/`):
Extract Scope from Controller, Replace Conditional with Polymorphism,
Replace Conditional with Null Object, Introduce Form Object, Replace
Subclasses with Strategies, Replace Mixin with Composition, Extract Validator,
Introduce Parameter Object, Replace Callback with Method, Refactor Service
Object into Domain Model.

## Tech Stack

These skills are opinionated. The conventions they enforce:

| Concern | Choice |
|---------|--------|
| Testing | Minitest with fixtures |
| Authorization | Pundit |
| State machines | AASM |
| Notifications | Noticed |
| Components | Phlex |
| Background jobs | Solid Queue |
| Event pipeline | Rails.event (8.1+) |
| Deployment | Kamal + Thruster |
| Frontend | Hotwire (Turbo + Stimulus) |

## License

MIT
