# Changelog

All notable changes to the public jr-rails-skills marketplace are documented
here. This changelog is used by `scripts/sync-public.sh` to populate GitHub
release notes on julianrubisch/skills.

## Unreleased

## v1.2.0

- Add **jr-rails-bootstrap** skill: one-time Mac bootstrap from a blank
  dev environment to a Rails app running in the browser. Conversational
  interview + automated execution, idempotent prereq checks (Xcode CLT,
  Homebrew, mise, Ruby 4.0, Node 22), git hosting CLI setup
  (gh and/or glab) with a separate-Terminal-tab auth handoff, skills
  pack install, and delegation to jr-rails-new for app generation.
  Pedagogical voice for non-technical users. Writes
  `SETUP_SUMMARY.md` as an audit trail.
- **jr-rails-new** improvements wired in alongside bootstrap:
  - Add **Step 0: Preflight**. Probes ruby/node/bundle/mise versions
    and offers to hand off to `/jr-rails-bootstrap` if anything is
    missing or out of date, instead of failing at `rails new` with a
    confusing error.
  - **CLAUDE.md template** now includes a load-bearing
    `## How to start` section, derived from the actual stack choices
    (default / vite / devcontainer / worktree). `/jr-rails-bootstrap`
    Phase G reads this to know how to launch the dev server.
  - **Final summary** prints a start command that matches the
    generated CLAUDE.md instead of hardcoding `bin/dev`.

## v1.1.2

- Harden CLI discovery in jr-rails-second-opinion. Don't trust `$PATH`
  or `$SHELL` (the latter often reports `/bin/zsh` for macOS users who
  actually run fish via `exec fish` from `.zshrc`). Probe every
  installed user shell (fish, zsh, bash), merge their login PATHs,
  and augment with common install dirs (asdf/mise shims,
  npm/bun/pnpm/cargo/deno global bins, `/opt/homebrew/bin`,
  `~/.local/bin`). Reuse the resolved PATH when invoking the chosen
  CLI. On 0 detections, ask the mediator for an explicit binary path
  before suggesting an install.

## v1.1.1

- Add `claude` (Anthropic Claude Code CLI) as a supported reviewer in
  jr-rails-second-opinion. Useful when the skill is invoked from a non-Claude
  harness (codex, opencode, etc.) and Claude itself is the external lens.

## v1.1.0

- Add **jr-rails-second-opinion** skill: get a Rails-flavored second opinion on
  a branch, PR, or working tree by delegating to a locally-installed agentic
  CLI (codex, opencode, gemini, aider, mods, cursor-agent, llm, goose). Probes
  `$PATH` first, asks the user which CLI to use, then runs a structured
  Self-Review → External Review → Reconcile → Synthesize loop with H/M/L
  severity gating. Self-contained: no MCP server required.

## v1.0.1

- Add "Type-Checking Dispatch" smell with detection signals and decision table
  (concern vs presenter vs delegated_type)
- Extend refactoring 002 (Replace Conditional with Polymorphism) with "Where to
  Put the Extracted Behavior" — two shapes: controller→presenter,
  model→concern/delegated_type
- Add "The Abstraction Ladder" to architecture Rule 2 — model→presenter→
  component→controller with "push it down" guidance
- Add LSP and ISP to complete SOLID principles coverage
- Fix marketplace.json schema for Claude Code /plugin install

## v1.0.0

Initial public release of jr-rails-skills marketplace.

### Skills

- **jr-rails-classic** — Write Rails code in 37signals/classic style: rich
  models, CRUD controllers, concerns, state-as-records, Minitest
- **jr-rails-new** — Scaffold a new Rails app with preferred stack: PostgreSQL,
  Minitest, Solid Queue, Pundit, optional Phlex, optional agentic worktree setup
- **jr-rails-phlex** — Write Phlex views and components for Rails: class
  hierarchy, slots, helpers, custom elements

### Reference Docs (bundled with each skill)

- Architecture patterns, anti-patterns, code smells
- Coding style guides (classic + Phlex)
- Design principles (SOLID, Tell Don't Ask, Law of Demeter)
- Agentic worktree workflow (SQLite, PostgreSQL, devcontainer strategies)
- DevOps/Kamal deployment patterns
