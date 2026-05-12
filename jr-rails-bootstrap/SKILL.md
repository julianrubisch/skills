---
name: jr-rails-bootstrap
description: >-
  One-time Mac bootstrap from blank dev environment to a Rails app running in
  the browser. Installs Xcode CLT, Homebrew, mise, Ruby, Node, gh/glab, the
  jr-rails skills pack, then delegates to jr-rails-new (or wires up an existing
  app). Pedagogical voice for non-technical users.
triggers:
  - /jr-rails-bootstrap
  - jr-rails-bootstrap
  - bootstrap my mac for rails
  - set up rails on a new mac
  - new mac rails setup
  - i have a new mac and want to build a rails app
  - help me install rails
  - i'm new to rails, help me start
  - first rails app on this machine
---

# Rails Bootstrap (One-time Mac Setup)

Take a Mac user from a near-blank dev environment to a Rails app running in their browser. Conversational interview, automated execution, pedagogical voice. **Delegates** Rails app generation to `jr-rails-new`; this skill handles the prerequisites and the wiring.

Read `@references/guide.md` and follow it. Do not proceed without it.

## When to use

- Brand-new Mac with no dev tools installed.
- First Rails app on this machine.
- User is non-technical and wants a guided path.

**Not for:** generating additional Rails apps after the machine is set up (use `jr-rails-new` directly), per-project work (use `jr-rails-classic` / `jr-rails-phlex`), deployment (out of scope), or SSH key setup (out of scope - HTTPS only).

## Hard preconditions (assumed, not verified)

- Mac on macOS 14+
- Claude Code already installed, authenticated, running (this skill is invoked **from** Claude Code; it cannot install its host)
- Claude Pro or Max subscription (no API-key spend assumed)

If any of these is missing the user cannot reach this skill at all - so no defensive check.

## Phase summary

| Phase | What | Output |
|-------|------|--------|
| A | Sanity (macOS version, subscription) + fresh-or-existing branch | `BRANCH`, optional `PROJECT_PATH` |
| B | Prereqs: Xcode CLT, Homebrew, mise, Ruby 4.0, Node 22 | tool chain ready |
| C | Git hosting CLI(s): gh, glab. **Auth happens in a separate Terminal window** | gh/glab authenticated on host shell |
| D | Skills pack via `npx skills add julianrubisch/skills -g -y` | jr-rails-* available |
| E | App: `jr-rails-new` (fresh branch) or smoke-check (existing branch) | working Rails project |
| F | Git remote: create new or attach existing | `origin` set, first push (fresh) |
| G | Run the app: read project's `CLAUDE.md`, fall back to `bin/dev`, open browser | "It works" |

Full per-step detail (commands, prompts, error messages, pedagogical "why this matters" lines) lives in `references/guide.md`. Load it at the start of a session and step through it.

## Hard rules

**Idempotent everywhere.** Every prereq check is "✓ already installed (version X)" if present. Every auth check is "✓ already authenticated as Y". Re-running on a configured Mac must complete without side effects beyond producing a fresh `SETUP_SUMMARY.md`.

**Never run interactive auth flows inside the Claude Code session.** `gh auth login` and `glab auth login` write credentials to the host shell's keychain/config. If you run them inside the Claude Code Bash tool, the credentials may land in the wrong context or get truncated by tool output buffering. Instruct the user to open a fresh Terminal window and run the auth command there. Wait for explicit confirmation before proceeding. See `references/auth-handoff.md` for the exact script.

**Existing-app branch is read-only on user files.** Never overwrite their project, never `rails new` over it, never `git init`, never run a destructive migration. The only write into an existing project's tree is `git remote add origin` (Phase F), and only if no remote is currently set.

**Visible checklist.** Print a running progress line at every step: `Step 6 of 20: installing mise...`. The user must be able to see where in the process they are.

**Pause between phases.** Confirm with the user before crossing a phase boundary. Skip the pause only if the user opts into non-interactive ("run all") mode at the start.

**Plain-language errors.** Never paste a raw stack trace at the user. Translate: "Homebrew install failed because `/usr/local` is owned by another user. Fix: run `sudo chown -R $(whoami) /usr/local` in Terminal." Show the underlying error after the explanation if needed, not before.

**Final artifact.** Always write `<project>/SETUP_SUMMARY.md` at the end (template in `references/setup-summary-template.md`). This is the audit trail; future invocations can read it instead of re-detecting.

**Voice.** The user is non-technical. One short "why this matters" sentence before each install step. Reassurance during slow steps (e.g. compiling Ruby). No jargon without a one-line definition. **Use hyphens, not em-dashes** in any prose written for the user.

## File layout

```
jr-rails-bootstrap/
├── SKILL.md                              # This file (entry point)
└── references/
    ├── guide.md                          # Full phase-by-phase walkthrough (load first)
    ├── auth-handoff.md                   # Phase C step 10 script for separate-shell auth
    └── setup-summary-template.md         # Final artifact template
```

Progressive disclosure: only load reference files as you need them. The entry point stays cheap.
