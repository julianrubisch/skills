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
| `jr-rails-second-opinion` | Get a Rails-flavored second opinion on a branch, PR, or working tree by delegating to a locally-installed agentic CLI (codex, opencode, gemini, aider, mods, …). Self-contained: no MCP server required |
| `jr-rails-pr-tour` | Turn a large PR or MR into a guided reading order: chapters of related files ordered by Rails layer and dependency, a one-line "why here" per file, churn × complexity risk badges, and the routes to open on a dev server. Published as a page that drives the diff in a second tab. GitHub and GitLab |
| `jr-rails-bootstrap` | One-time Mac bootstrap from a blank dev environment to a Rails app running in the browser: Xcode CLT, Homebrew, mise, Ruby, Node, gh/glab, this skills pack, then `jr-rails-new`. Written for non-technical users |

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

## jr-rails-second-opinion

Delegates a Rails-flavored review to a locally-installed agentic CLI. Wraps
the chosen CLI in a structured Self-Review → External Review → Reconcile →
Synthesize loop with H/M/L severity gating. Self-contained: no MCP server.

**Invoke:**

- `/jr-rails-second-opinion` (review uncommitted changes; default)
- `/jr-rails-second-opinion <branch>` (review the branch diff vs main)
- `/jr-rails-second-opinion <PR#>` (fetch the PR with `gh`, review the diff)
- `/jr-rails-second-opinion --multi` (run review through 2+ CLIs in parallel and reconcile)

Phase 0 is CLI discovery. The skill probes `$PATH` for known agentic CLIs and
asks you which to use. Default detection list:

`codex` · `opencode` · `gemini` · `aider` · `mods` · `cursor-agent` · `llm` · `goose`

Per-CLI invocation patterns live in `references/cli-invocations.md` and are
edit-friendly (extend with new CLIs as they ship).

The brief passed to the chosen CLI bakes in jr-rails-skills' Rails-flavored
review dimensions (architecture, quality, performance, testing, security)
plus the hard rule against service-object suggestions. Output is a working
log (`second-opinion.md`) with rounds, reconciliation tables, gate status,
mediator approval, and final attestation. Deleted after attestation; the
improved artifact is the deliverable, not the log.

Findings are judged against the pack's own standards, not the CLI's taste:
`references/standards.md` lists them per dimension with links into the
shared references, its brief block goes into every CLI brief, and Reconcile
drops CLI findings that contradict a standard (service objects, factories,
mail in a callback) with the standard cited.

## jr-rails-pr-tour

Turns a large PR or MR into a reading order. Code review tools list files
alphabetically; reviewers report more context switching and missed bugs
that way, and ask for dependency order and grouping by purpose instead.
Rails makes both cheap: the layer is in the path, the dependencies are in
the constants.

**Invoke:**

- `/jr-rails-pr-tour <PR or MR URL>`
- `/jr-rails-pr-tour <number>` (host inferred from `git remote`)
- `/jr-rails-pr-tour` (current branch vs `main`, no host links)
- `--comment` also posts the tour as a PR/MR comment for co-reviewers

**What it produces:** a page with chapters of related files in the order to
read them. Per file: a one-line "why here", a "look for" line where
something deserves attention, and a risk badge that measures the change
(complexity the PR added, from `attractor diff`; how many of the PR's
commits touched the file), not the code's history. Every link opens the
diff at that file in a second, reused browser tab, via GitHub (sha256) and
GitLab (sha1) file anchors computed in the page. Checkboxes remember what
you have read. Optional: a churn × complexity scatter of the touched files
with base → head traces (linear/log), and the routes to open on a running
dev server, per chapter, with the project's devcontainer started for you
when it has one.

**How the order is derived:** noise filter (lockfiles, schema dump,
fixtures, build output; rendered as a skip list so nothing disappears),
Rails layer per file, symbol references between changed files plus Rails
conventions (associations, controller → views, routes → controllers,
migration → model, test → subject), union-find clustering, topological
order inside a chapter, then a judgement pass that names the chapters and
reshapes what the mechanics got wrong. Prose follows Simplified Technical
English. It is a map, not a review; it does not produce findings.

**Requires:** `gh` or `glab` authenticated for host links and `--comment`.
`attractor` with `attractor-ruby` (and `attractor-javascript`) for
complexity badges and the scatter; the skill finds it under any installed
Ruby, installs it when none has it, and falls back to `flog`, then to diff
size. GitHub, GitHub Enterprise, gitlab.com and self-hosted GitLab.

## jr-rails-bootstrap

One-time setup of a Mac from a near-blank dev environment to a Rails app
running in the browser. Written for non-technical users: a conversational
interview, automated execution, one short "why this matters" line before
each install step, plain-language errors, and a visible
`Step 6 of 20: installing mise...` progress line throughout.

**Invoke:**

- `/jr-rails-bootstrap`

**Phases:**

| Phase | What |
|-------|------|
| A | Sanity check (macOS version, subscription); fresh app or existing app? |
| B | Prerequisites: Xcode CLT, Homebrew, mise, Ruby 4.0, Node 22 |
| C | Git hosting CLI(s): `gh` and/or `glab`; auth happens in a separate Terminal window |
| D | This skills pack via `npx skills add julianrubisch/skills -g -y` |
| E | The app: `jr-rails-new` (fresh) or a smoke check (existing) |
| F | Git remote: create new or attach existing; first push |
| G | Run the app (`bin/dev` or the project's `CLAUDE.md` instructions) and open the browser |

Every step is idempotent ("✓ already installed (version X)"), so re-running
on a configured Mac completes without side effects. Interactive auth never
runs inside the Claude Code session; the skill hands you the command for a
fresh Terminal window and waits. An existing app's files are never
overwritten. The audit trail is `SETUP_SUMMARY.md` in the project, which a
later run reads instead of re-detecting.

**Assumes:** macOS 14+, Claude Code installed and authenticated, a Claude
Pro or Max subscription. Deployment and SSH keys are out of scope.

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
