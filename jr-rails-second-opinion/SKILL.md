---
name: jr-rails-second-opinion
description: >-
  Get a Rails-flavored second opinion on a branch, PR, or working tree by
  delegating to a locally-installed agentic CLI (claude, codex, opencode,
  gemini, aider, etc.). Self-contained: no MCP server required.
triggers:
  # Direct invocations
  - /jr-rails-second-opinion
  - jr-rails-second-opinion
  - rails second opinion
  - rails sparring partner
  # Specific CLI hooks
  - review my Rails branch with codex
  - review my Rails branch with opencode
  - codex review my branch
  - opencode review my branch
  - use codex to review
  - use opencode to review
  - use gemini to review
  - use aider to review
  - use claude to review
  # Generic phrasing
  - second opinion on this branch
  - external Rails review
  - have another agent review this
  - get another model to review
  - review my Rails work with another CLI
  # Convergence / refinement
  - dual-CLI review
  - multi-CLI review
  - cross-CLI sparring
---

# Rails Second Opinion (Local Agentic CLI)

Delegate review of Rails work (uncommitted diff, branch, or PR) to a locally-installed agentic CLI. Wraps the CLI in a structured Self-Review → External Review → Reconcile → Synthesize loop with H/M/L severity gating, mirroring the consult-outside-expert workflow but without MCP dependence.

Read `@references/guide.md` and follow it. Do not proceed without it.

## Invocation

- `/jr-rails-second-opinion` (review uncommitted changes; default)
- `/jr-rails-second-opinion <branch>` (review `<branch>` diff vs main)
- `/jr-rails-second-opinion <PR#>` (fetch PR with `gh`, review the diff)
- `/jr-rails-second-opinion --multi` (run review through 2+ CLIs in parallel and reconcile)

## Workflow Overview

### 1. CLI Discovery (Required First Step)

Probe for installed agentic CLIs. **Do not trust the inherited `$PATH` or `$SHELL`** — Claude Code's Bash tool spawns a non-login shell that misses fish/zsh PATH entries and most version-manager shims (asdf, mise, volta, fnm, bun, pnpm). Worse, `$SHELL` may report `/bin/zsh` even when the user actually runs fish (macOS user record defaults to zsh; fish is exec'd from `.zshrc`). Probe every installed user shell, merge their login PATHs, then augment with common install dirs:

```bash
# Don't trust $SHELL — on macOS it often reflects the user-record default (zsh)
# even when the user actually runs fish via `exec fish` from .zshrc. Probe
# every installed user shell and merge their login PATHs.
PATHS_TO_TRY=()
for sh in fish zsh bash; do
  command -v "$sh" >/dev/null 2>&1 || continue
  case "$sh" in
    fish) p=$("$sh" -l -c 'string join : -- $PATH' 2>/dev/null) ;;
    *)    p=$("$sh" -l -c 'printf %s "$PATH"' 2>/dev/null) ;;
  esac
  [ -n "$p" ] && PATHS_TO_TRY+=("$p")
done
LOGIN_PATH=$(IFS=:; echo "${PATHS_TO_TRY[*]}")
[ -z "$LOGIN_PATH" ] && LOGIN_PATH="$PATH"

EXTRA_DIRS="$HOME/.local/bin:$HOME/.npm-global/bin:$HOME/.bun/bin:$HOME/.cargo/bin:$HOME/.deno/bin:$HOME/.asdf/shims:$HOME/.mise/shims:$HOME/Library/pnpm:/opt/homebrew/bin:/usr/local/bin"
SEARCH_PATH="$LOGIN_PATH:$EXTRA_DIRS"

for cmd in claude codex opencode gemini aider mods cursor-agent llm sgpt goose; do
  bin=$(PATH="$SEARCH_PATH" command -v "$cmd" 2>/dev/null) || continue
  echo "$cmd -> $bin"
done
```

Then call `AskUserQuestion` with the detected CLIs as options:

- If 0 found, **ask the mediator before escalating to install**. They may have a CLI at a non-standard path — accept an absolute path via the "Other" answer. Only suggest installing (`brew install codex`, `npm i -g opencode-ai`) after they confirm nothing is installed.
- If 1 found, still confirm it as the choice.
- If 2+ found, list them as options. Add an "all of them (multi mode)" option when `--multi` is passed.

Reuse the resolved `$SEARCH_PATH` (or the absolute path) when invoking the CLI later — the same shell-context issue applies to invocation, not just detection.

### 2. Scope

Determine review scope:

- **Default** (no arg): uncommitted changes (`git diff` + untracked summary).
- **Branch**: `git diff <base>...<branch>` where base is `main` (or current upstream).
- **PR#**: `gh pr checkout <num>` then branch flow, or fetch the PR diff via `gh pr diff`.

### 3. Brief Construction

Build the brief passed to the CLI. Force the H/M/L severity rubric and the five Rails-flavored review dimensions: architecture (skinny controllers, rich models, callback design), quality (Ruby idiom, naming, anti-patterns), performance (N+1, indexes, eager loading), testing (coverage, integration/unit balance), security (strong params, mass assignment, Brakeman class issues). Include the project's hard rule: **never recommend service objects as a solution**.

See `references/guide.md` for the full brief template.

### 4. Run Review

Invoke the chosen CLI with its review-style command. Per-CLI invocations live in `references/cli-invocations.md`. Capture stdout to the working log.

### 5. Self-Review + Reconcile + Synthesize

Implementer (you) does a parallel self-review, then reconciles with the CLI's findings. Produce a synthesis with Consensus / Disagreements / Actions / Gate Status. Wait for mediator approval before acting.

### 6. Act + Iterate

Address agreed actions. Open a Round 2 if needed (re-run CLI on the updated artifact, capture deltas only). Stop on convergence: no open H, all M either fixed or accepted.

### 7. Cleanup

Delete the working log (`second-opinion.md`) after final attestation. The improved artifact is the deliverable, not the log.

## Hard Rules

**Inherit from jr-rails-review:** never recommend service objects (`*Service`, `*Manager`, `*Handler`, `*Processor`, `*Creator`, `.call` PORO patterns). Pass this rule into the second-opinion CLI's brief so it doesn't suggest them either. When extraction is needed, prefer domain models, form objects, query objects, concerns, DCI contexts, or callback extraction.

**Never act on a CLI finding without mediator approval.** The CLI is a signal generator, not a judge. Decisions stay with the human.

**Do not pipe secrets to the CLI.** Brief construction must scrub `.env`, `config/credentials/*`, and any file Brakeman flags as containing secrets.

## File Layout

```
jr-rails-second-opinion/
├── SKILL.md                       # This file (entry point)
└── references/
    ├── guide.md                   # Workflow detail (load when starting a session)
    ├── cli-invocations.md             # Per-CLI invocation patterns + flags (load when picking a CLI)
    └── second-opinion-log-template.md # Working-log skeleton (copy when starting a session)
```

Load reference files only when you need them. Progressive disclosure keeps the entry point cheap.
