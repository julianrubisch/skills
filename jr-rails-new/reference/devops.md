# DevOps / Infrastructure Reference

## Deployment

**Kamal is the default.** Ships with Rails 8+, deploys Docker containers to
any VPS via SSH. Zero-downtime rolling deploys, no orchestration layer needed.
Just a server and a container registry.

### Kamal (Default)

```bash
kamal setup          # first deploy: provision + push + start
kamal deploy         # subsequent deploys
kamal app exec rails console  # remote console
```

Kamal handles the full lifecycle: builds the Docker image, pushes to a registry,
pulls on the target server, runs health checks, and swaps traffic with zero
downtime via Traefik. Pairs with Thruster for HTTP/2 and asset caching.
Kamal 2 ships with a built-in local registry — no external registry service
needed for single-server deployments.

**When to choose:** most Rails apps. One to many servers, single or multi-service.

### PaaS (Render, Fly.io, Railway)

Managed platforms that handle the runtime, registry, TLS, and auto-scaling.
Trade cost for operational simplicity.

**When to choose:** MVPs, small teams that don't want to manage servers,
or apps where deployment is not a core competency.

### Kubernetes

Full cluster orchestration. Maximum control and scaling, but requires
significant DevOps expertise. Use managed K8s (EKS, GKE, AKS) to reduce
the burden.

**When to choose:** only when the org already runs K8s or genuinely needs
auto-scaling across dozens of services.

### Decision Heuristic

| Scenario | Recommendation |
|---|---|
| Most Rails apps | **Kamal** on 1+ servers |
| MVP, no ops capacity | PaaS |
| Large org with platform team | Kubernetes |

## Development Environment

**Prefer dev containers** to isolate workloads and eliminate "works on my machine"
problems. Every project should ship a `.devcontainer/` directory.

### Setup

Start with `rails new --devcontainer` (Rails 7.2+) to scaffold the boilerplate.
Three files in `.devcontainer/`:

- **`devcontainer.json`** — features (Ruby, Node, PG client), extensions,
  env vars, `postCreateCommand` for `bin/setup`
- **`compose.yaml`** — services (app, database, Selenium for system tests),
  volume mounts, port mappings
- **`Dockerfile`** — minimal, extends the official Rails base image

### Key Patterns

- Mount SSH keys as **read-only bind mounts** for seamless Git operations
  (e.g. `/home/vscode/.ssh/id_ed25519:ro`). Use a custom SSH config inside
  the container to avoid macOS-specific directives (`UseKeychain`) breaking
  on Linux.
- `sleep infinity` in compose so containers stay up — the app is not
  auto-started, giving you flexibility in when/how you launch it
- Puma binds to `0.0.0.0` (not `127.0.0.1`) for port accessibility.
  Especially critical in Codespaces: `bin/dev -b 0.0.0.0`.
- Database as a separate compose service with persistent volumes
  (`postgres_data:/var/lib/postgresql/data` to survive restarts)
- Use `:cached` consistency mode on workspace volume mounts for faster reads

### compose.yaml Details

- **Selenium service:** add `selenium/standalone-chromium` for system tests.
  Set `SELENIUM_HOST=selenium` and `CAPYBARA_SERVER_PORT=45678` in
  devcontainer.json env vars so Capybara connects correctly.
- **Port forwarding with CLI:** `devcontainer-cli` does not support automatic
  port forwarding. Add explicit `ports: ["3000:3000"]` to compose.yaml when
  using CLI tools instead of VS Code.

### devcontainer.json Features

Useful features beyond Ruby and Node:

- `docker-outside-of-docker` — required for Kamal deployments using local builders
- ImageMagick / FFmpeg — for ActiveStorage image/video processing
- `postgres-client` — connection libraries only (no server)
- Third-party features (e.g. `stripe-cli`) can be stacked via JSON declarations

For complex setup, replace `postCreateCommand` with a `.devcontainer/boot.sh`
script that handles multiple steps (cleanup of leftover socket files, etc.).

### Dockerfile

Use the Rails-provided base images (`ghcr.io/rails/devcontainer/images/ruby:$RUBY_VERSION`)
rather than standard Ruby images — they are optimized for dev containers and
support feature stacking. This Dockerfile is strictly for dev, not production.

### Execution

- **VS Code / Cursor** — auto-detects `.devcontainer/`, offers to rebuild
  when config changes
- **`devcontainer-cli`** — standalone CLI for headless / CI use. Requires
  explicit `--workspace-folder`; use `--no-cache` to rebuild from scratch.
- **GitHub Codespaces** — cloud-based, zero local setup. Auto-detects running
  ports and prompts to open them in browser tabs.

Reference: https://www.rorvswild.com/blog/2025/dev-containers-rails

## Agentic Worktree Workflow

See [agentic-worktrees.md](agentic-worktrees.md) for isolating concurrent
AI agent sessions with per-workspace databases, containers, and port ranges.

## Feature Flags

Use **Flipper** to decouple deployment from release. Feature flags enable
trunk-based development, gradual rollouts, and A/B testing without redeployment.

### Setup

```bash
bundle add flipper flipper-active_record flipper-ui
bin/rails g flipper:setup
bin/rails db:migrate
```

Mount the admin UI for non-technical stakeholders:

```ruby
# config/routes.rb
mount Flipper::UI.app(Flipper) => "/flipper"
```

### Gate Types

**Boolean (global on/off):**
```ruby
Flipper.enable(:new_checkout)
Flipper.enabled?(:new_checkout)  # => true for everyone
```

**Individual actors** — target specific users. Actors must respond to
`flipper_id` (ActiveRecord default: `"User;42"`):
```ruby
Flipper.enable(:new_checkout, User.find(42))
Flipper.enabled?(:new_checkout, current_user)
```

**Groups** — register reusable segments in an initializer:
```ruby
# config/initializers/flipper.rb
Flipper.register(:admins) do |actor, _context|
  actor.respond_to?(:role) && actor.role == "admin"
end

Flipper.enable_group(:new_checkout, :admins)
```

**Percentage of actors** — consistent per user (same users always see it).
Use for gradual rollouts:
```ruby
Flipper.enable_percentage_of_actors(:new_checkout, 10)  # 10% of users
Flipper.enable_percentage_of_actors(:new_checkout, 50)  # ramp up
Flipper.enable(:new_checkout)                            # full rollout
```

**Percentage of time** — random per request, inconsistent per user. Avoid
for user-facing features:
```ruby
Flipper.enable_percentage_of_time(:cache_experiment, 25)
```

### Integration Patterns

**Controller guard:**
```ruby
class LikesController < ApplicationController
  def create
    return head(:not_found) unless Flipper.enabled?(:likes, current_user)
    # ...
  end
end
```

**Authorization integration** — consolidate flag checks into your authorization
layer to avoid scattering `Flipper.enabled?` calls:
```ruby
# app/policies/like_policy.rb
class LikePolicy < ApplicationPolicy
  def create?
    Flipper.enabled?(:likes, user)
  end
end
```

**Route constraint:**
```ruby
class FlipperConstraint
  def initialize(feature) = @feature = feature

  def matches?(request)
    user = request.env["warden"]&.user(:user)
    user && Flipper.enabled?(@feature, user)
  end
end

# config/routes.rb
resources :likes, constraints: FlipperConstraint.new(:likes)
```

**View conditional:**
```erb
<% if Flipper.enabled?(:new_checkout, current_user) %>
  <%= render "checkout_v2" %>
<% end %>
```

### Measuring Rollout Impact

Tag APM metrics with the flag state to compare enabled vs disabled cohorts.
Most APM tools (AppSignal, Datadog, New Relic, Scout) support custom tags
on distribution/counter metrics.

```ruby
# Tag with flag state — works with any APM that supports custom metrics
flag_on = Flipper.enabled?(:new_checkout, current_user)
duration = Benchmark.realtime { yield }

# Your APM's custom metric API, e.g.:
YourApm.distribution("checkout_duration", duration * 1000, tags: { new_checkout: flag_on })
YourApm.increment("checkout_completed", tags: { new_checkout: flag_on })
```

Capture flag state early in the request (store in an instance variable) to
ensure consistent evaluation between execution and telemetry.

### Best Practices

- **Naming:** use clear, semantic names (`:article_likes` not `:feature_123`)
- **Placement:** funnel flag checks through Pundit policies rather than
  scattering across controllers and views
- **Cleanup:** once a flag is 100% enabled and stable, remove the flag check
  and dead code path. Track stale flags via Flipper instrumentation events.
- **Performance:** Flipper memoizes per request cycle. For ActiveRecord
  adapter, ensure the `flipper_features` / `flipper_gates` tables are indexed.
- **Testing:** use `Flipper.enable(:flag)` in test setup; reset in teardown.
  Flipper's test adapter keeps state in memory.

References:
- https://blog.cloud66.com/how-to-add-feature-flags-to-your-ruby-on-rails-applications

## Configuration Management

For most apps, `Rails.application.credentials` or `ENV.fetch("KEY")` is enough.
When settings grow to 10+ values across 3+ services, or for multi-tenant /
self-hosted apps, consider structured config objects with boot-time validation.

> See `shared/configuration.md` for the Anyway Config pattern, YAML source
> precedence, and when the abstraction earns its keep.

## Performance Benchmarking

### Load Testing

Use `oha`, `wrk`, or `k6` for HTTP load testing. Run regularly, not just
before launches. Candidate for CI integration.

```sh
# oha — realistic admin backend scenario (2 req/s, 20s, no keepalive)
oha -q 2 -z 20s --disable-keepalive --latency-correction http://localhost:3000/posts

# wrk — higher throughput scenario
wrk -t4 -c100 -d30s http://localhost:3000/posts
```

Key metrics to track: mean response time, p95, p99. Chart over time to catch regressions.

### Application Load Time

Use `bumbler` to identify slow requires at boot. Target: keep added boot time
from any single gem under ~50ms.

```sh
gem install bumbler
bumbler -t 50   # flag requires slower than 50ms
```

Compare against lightweight equivalents (e.g. Devise ~20ms, Ransack ~20ms)
as a calibration benchmark.

### Memory Store for Short-lived Application Data

A small `ActiveSupport::Cache::MemoryStore` is useful for caching data that's
expensive to compute, stable for the lifetime of a worker, and small enough
to fit in memory (e.g. file contents, computed hashes).

```ruby
# config/initializers/resource_registry.rb
module ResourceRegistry
  CACHE = ActiveSupport::Cache::MemoryStore.new(size: 2.megabytes)

  def self.fetch(key, &block)
    CACHE.fetch(key, &block)
  end
end

# Usage
ResourceRegistry.fetch(file_path) { File.read(file_path) }
```

## Production Hardening

### Puma Configuration

Rails generates a reasonable default `config/puma.rb`. For production, ensure
`preload_app!` is enabled for copy-on-write memory savings with workers:

```ruby
# config/puma.rb
max_threads_count = ENV.fetch("RAILS_MAX_THREADS", 5)
min_threads_count = ENV.fetch("RAILS_MIN_THREADS", max_threads_count)
threads min_threads_count, max_threads_count

port ENV.fetch("PORT", 3000)
environment ENV.fetch("RAILS_ENV", "development")

workers ENV.fetch("WEB_CONCURRENCY", 2)

preload_app!
```

**Key settings:**
- `preload_app!` — loads the app before forking workers (CoW-friendly, faster
  boot). Trade-off: can't use phased restarts.
- `WEB_CONCURRENCY` — number of worker processes. Start with 2, tune based on
  available memory (each worker ≈ app memory footprint).
- `RAILS_MAX_THREADS` — threads per worker. Match to database pool size.

### Database Production Config

Harden PostgreSQL connections with timeouts and pool management:

```yaml
# config/database.yml
production:
  adapter: postgresql
  pool: <%= ENV.fetch("RAILS_MAX_THREADS", 5) %>
  timeout: 5000
  reaping_frequency: 10
  connect_timeout: 2
  variables:
    statement_timeout: '30s'
```

**Key settings:**
- `statement_timeout` — kills queries running longer than 30s. Prevents
  runaway queries from holding connections.
- `connect_timeout` — fail fast if DB is unreachable (2s instead of default ~60s).
- `reaping_frequency` — how often to check for dead connections (seconds).
- `pool` — must match `RAILS_MAX_THREADS` to avoid connection exhaustion.

### SSL / HSTS

```ruby
# config/environments/production.rb
config.force_ssl = true
config.ssl_options = {
  hsts: {
    subdomains: true,
    preload: true,
    expires: 1.year
  }
}
```

### Dockerfile and CI

Rails 7.1+ generates a production-optimized Dockerfile (multi-stage build,
jemalloc, non-root user, Bootsnap precompilation) and a GitHub Actions CI
workflow (security scans, lint, tests, system tests). Use the scaffold defaults
— don't hand-roll these.

```bash
# Generated automatically by rails new:
# Dockerfile          — multi-stage production build
# .github/workflows/ci.yml  — CI with brakeman, rubocop, minitest
# .dockerignore       — excludes dev/test artifacts
```

