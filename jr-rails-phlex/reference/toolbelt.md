# Gem Toolbelt

Categorized gem recommendations across projects. Pattern-specific gems are
also referenced in their respective sections of `patterns.md` and `shared/`.

## Frontend

| Gem | Purpose |
|-----|---------|
| [turbo-rails](https://github.com/hotwired/turbo-rails) | Turbo Drive, Frames, Streams — SPA-like navigation without JS (see `shared/hotwire.md`) |
| [stimulus-rails](https://github.com/hotwired/stimulus-rails) | Modest JS framework for Hotwire apps (see `shared/hotwire.md`) |
| [phlex-rails](https://github.com/phlex-ruby/phlex-rails) | Ruby component framework replacing ERB (see `coding-phlex.md`) |
| [phlex_custom_element_generator](https://github.com/konnorrogers/phlex_custom_element_generator) | Auto-generate Phlex wrappers from custom element manifests |
| [vite_rails](https://github.com/ElMassimo/vite_ruby) | Vite integration for Rails — fast HMR, modern bundling |
| [propshaft](https://github.com/rails/propshaft) | Vanilla Rails asset pipeline (alternative to vite_rails) |
| [importmap-rails](https://github.com/rails/importmap-rails) | Import maps — no bundler needed for simple JS |
| [active_link_to](https://github.com/comfy/active_link_to) | Navigation helper with automatic active-state detection (inclusive, exclusive, regex matching) |

## Authorization & Identity

| Gem | Purpose |
|-----|---------|
| [pundit](https://github.com/varvet/pundit) | Policy-based authorization (see `patterns.md § Policy Objects`) |

## State & Workflow

| Gem | Purpose |
|-----|---------|
| [aasm](https://github.com/aasm/aasm) | State machines with guards, callbacks, ActiveRecord integration (see `shared/state_machines.md`) |
| [flipper](https://github.com/flippercloud/flipper) | Feature flags — inline `if Flipper.enabled?(:feature)` |

## Data & Queries

| Gem | Purpose |
|-----|---------|
| [pagy](https://github.com/ddnexus/pagy) | Fast, lightweight pagination |
| [has_scope](https://github.com/heartcombo/has_scope) | Declarative param-to-scope mapping in controllers (see `patterns.md § Filter Objects`) |
| [arel-helpers](https://github.com/camertron/arel-helpers) | Reduce Arel boilerplate for complex JOINs |
| [store_model](https://github.com/DmitryTsepelev/store_model) | Active Model interfaces for JSON/JSONB store attributes — typed, validatable, nestable (see `patterns.md § Value Objects`) |
| [frozen_record](https://github.com/byroot/frozen_record) | Query static YAML/JSON files with an ActiveRecord-like interface — ideal for lookup tables, configuration sets, and seed data that never changes at runtime |

## Serialization & Caching

| Gem | Purpose |
|-----|---------|
| [universalid](https://github.com/hopsoft/universalid) | Serialize any Ruby object (including AR models with unsaved changes) into compact, URL-safe strings — useful for passing complex state through URLs or preserving form state |
| [composite_cache_store](https://github.com/hopsoft/composite_cache_store) | Multi-layered cache combining fast in-process memory with slower shared remote caches — optimizes hot paths without custom plumbing |
| [active_model_serializers](https://github.com/rails-api/active_model_serializers) | DSL-based JSON serialization with associations — alternative to plain Ruby serializers when 10+ serializers share similar structure (see `shared/serializers.md`) |

## Notifications

| Gem | Purpose |
|-----|---------|
| [noticed](https://github.com/excid3/noticed) | Multi-channel notifications with database persistence (see `shared/notifications.md`) |

## Forms & Callbacks

| Gem | Purpose |
|-----|---------|
| [after_commit_everywhere](https://github.com/Envek/after_commit_everywhere) | `after_commit` callbacks outside ActiveRecord — used in form objects (see `patterns.md § Form Objects`) |

## Infrastructure & Ops

| Gem | Purpose |
|-----|---------|
| [solid_queue](https://github.com/rails/solid_queue) | Database-backed job queue — no Redis needed (see `shared/jobs.md`) |
| [mission_control-jobs](https://github.com/rails/mission_control-jobs) | Web UI for Solid Queue — monitor queues, retry failed jobs |
| [solid_cache](https://github.com/rails/solid_cache) | Database-backed cache store |
| [solid_cable](https://github.com/rails/solid_cable) | Database-backed Action Cable adapter |
| [kamal](https://github.com/basecamp/kamal) | Zero-downtime deploys via Docker |
| [thruster](https://github.com/basecamp/thruster) | HTTP/2 proxy with asset caching and compression for Puma |
| [autotuner](https://github.com/Shopify/autotuner) | Rack middleware that analyzes GC behavior at runtime and suggests concrete tuning parameters |
| [rack-attack](https://github.com/rack/rack-attack) | Request throttling and blocking — rate limiting for APIs (see `coding/api.md`) |

## Active Storage

| Gem | Purpose |
|-----|---------|
| [active_storage-blurhash](https://github.com/avo-hq/active_storage-blurhash) | Auto-generates BlurHash placeholders for images on upload — eliminates layout shift with tiny inline previews while full images load |

## Admin

| Gem | Purpose |
|-----|---------|
| [avo](https://github.com/avo-hq/avo) | Admin panel framework — resource CRUD, dashboards, custom tools, built on Hotwire |

## Development & Debugging

| Gem | Purpose |
|-----|---------|
| [model_probe](https://github.com/hopsoft/model_probe) | Schema visualization and code generation for ActiveRecord models — quick inspection of table structure and relationships |

## Gem Selection Heuristics

> From `coding-classic.md`:

1. **Can vanilla Rails do this?** — ActiveRecord, ActionMailer, ActiveJob cover most needs
2. **Is it the app's core concern?** — if yes, own the code; if fringe, use a gem
3. **Does it add infrastructure?** — Redis? Database-backed alternatives exist
4. **Is the complexity worth it?** — 150 lines of custom code vs. a 10k-line gem
5. **Is it from someone you trust?** — 37signals, Shopify, Heartcombo gems are battle-tested

> "Build solutions before reaching for gems."
