# Rails Bootstrap Guide

Full phase-by-phase walkthrough. The implementer (you) executes the steps; the mediator (the user) confirms phase boundaries. Use the pedagogical voice described in `SKILL.md`: one-line "why this matters" before each install, plain-language errors, hyphens not em-dashes.

---

## Setup

Before Phase A, ask once:

- "Want me to pause between phases for confirmation (recommended for a new Mac), or run straight through? Type **pause** or **run all**." Default: **pause**.

Print at the start:

```
Setting up Rails on this Mac. 20 steps across 7 phases.
Mode: pause between phases.
```

Maintain an internal step counter and print `Step N of 20: <action>` before every install or check.

---

## Phase A - Sanity + branch (steps 1-3)

### Step 1: macOS version

> *Why this matters: Rails 8 and its tooling assume modern Foundation libraries. macOS 14 (Sonoma) or newer is the supported floor.*

```bash
sw_vers
```

Parse `ProductVersion`. If the major version is less than 14:

```
Your Mac is on macOS <X>. Rails 8 needs macOS 14 (Sonoma) or newer.
Update via System Settings > General > Software Update, then re-run me.
Stopping here.
```

Abort. Do **not** offer to install anyway.

### Step 2: Subscription confirmation

Ask once using `AskUserQuestion`:

- Question: "Are you on Claude Pro or Claude Max? (This skill assumes no API-key spend.)"
- Options: `Yes (Pro)`, `Yes (Max)`, `No`.

If `No`: explain that the bootstrap will burn through API credits without a subscription, and stop. Don't continue under "they said they understand the cost" - just stop.

### Step 3: Fresh-or-existing branch

Ask via `AskUserQuestion`:

- Question: "Do you already have a Rails project on this Mac, or do you want a fresh one?"
- Options: `Fresh app (recommended)`, `I have an existing project`.

If existing, capture the absolute path:

- Follow-up question: "Paste the full path to the existing project."
- Validate that the path exists and is a directory before continuing. If not, re-ask up to twice, then offer to switch to the fresh-app branch.

Store the branch (`fresh` or `existing`) and `PROJECT_PATH` for use in Phase E.

**Pause boundary.** Summarise Phase A choices ("✓ macOS 14.4, ✓ Claude Max, branch: fresh") and confirm before Phase B.

---

## Phase B - Prereqs (steps 4-7)

Every step in this phase is idempotent: check first, install only if missing, report version after.

### Step 4: Xcode Command Line Tools

> *Why this matters: most native Ruby gems and Homebrew formulas compile C code. The CLT ships the compiler and headers.*

```bash
xcode-select -p 2>/dev/null
```

If exit code 0 and the path exists: `✓ Xcode Command Line Tools already installed`.

If absent:

```
Installing Xcode Command Line Tools. A GUI installer window will open.
This takes 10-15 minutes. You'll see lines scroll then pause - that's normal.

Click "Install" in the popup, then accept the licence. I'll wait.
```

Run `xcode-select --install`. The command returns immediately; the GUI installer keeps running. Poll:

```bash
until xcode-select -p >/dev/null 2>&1; do sleep 30; done
```

Wait up to 30 minutes. If still missing, ask the user to confirm they clicked Install. Don't kill the wait silently.

### Step 5: Homebrew

> *Why this matters: Homebrew is the package manager for everything else we need (mise, gh, glab).*

```bash
brew --version 2>/dev/null
```

If present: `✓ Homebrew already installed (<version>)`.

If missing:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After install, on Apple Silicon Macs Homebrew lives at `/opt/homebrew`. Append to `~/.zprofile` if not already present:

```bash
eval "$(/opt/homebrew/bin/brew shellenv)"
```

Then `eval "$(/opt/homebrew/bin/brew shellenv)"` in the current session so subsequent steps see `brew`.

### Step 6: mise

> *Why this matters: mise manages Ruby and Node versions per project. Cleaner than `rbenv` + `nvm` and pins versions in `.tool-versions`.*

```bash
mise --version 2>/dev/null
```

If present: `✓ mise already installed (<version>)`.

If missing:

```bash
brew install mise
```

Append to `~/.zshrc` if not already present:

```bash
eval "$(mise activate zsh)"
```

Then activate in the current session: `eval "$(mise activate zsh)"`.

### Step 7: Ruby + Node

> *Why this matters: Rails 8 needs Ruby 4.0+. Node 22 is the current LTS - used by importmap, esbuild, Vite, and the assets pipeline.*

```bash
ruby --version 2>/dev/null
node --version 2>/dev/null
```

If Ruby reports `ruby 4.0` or newer: `✓ Ruby already at <version>`. Else:

```bash
mise use --global ruby@4.0
```

Reassure during compile: "Your Mac is compiling Ruby. Lines will scroll then pause for a couple of minutes - that's normal."

Same pattern for Node:

```bash
mise use --global node@22
```

After both succeed:

```
✓ Ruby <version>
✓ Node <version>
```

**Pause boundary.** Confirm before Phase C.

---

## Phase C - Git hosting CLI(s) (steps 8-10)

### Step 8: Which platforms?

Ask via `AskUserQuestion` (multiSelect):

- Question: "Which git hosting platforms do you use? Pick all that apply."
- Options: `GitHub`, `GitLab`.

If the user picks neither, ask once more clarifying that at least one is required for Phase F. If they still refuse, skip Phase F entirely and warn that the project won't be pushed anywhere.

Store the selected set as `PLATFORMS`.

### Step 9: Install CLIs

For each platform in `PLATFORMS`:

| Platform | Check | Install |
|----------|-------|---------|
| GitHub | `gh --version` | `brew install gh` |
| GitLab | `glab --version` | `brew install glab` |

Report `✓ gh already installed (<version>)` or `Installed gh <version>` after.

### Step 10: Auth (the critical handoff)

**Stop and read `references/auth-handoff.md` before executing this step.** The auth flow runs in a **separate Terminal tab** (Cmd-T) - the user is already in Terminal, they just need a fresh shell that isn't running Claude Code.

Brief summary (full script in the handoff doc):

1. Check `gh auth status` / `glab auth status`. If exit code 0 and a username is reported: `✓ authenticated as <user>`. Skip the rest of step 10 for that platform.
2. If unauthenticated, instruct the user (with **bold warning**) to open a new Terminal window and run:
   - `gh auth login` (pick HTTPS, follow the prompts)
   - `glab auth login` (pick HTTPS)
3. Wait for the user to type `done` (or equivalent) before re-checking auth status.
4. Re-verify with `gh auth status` / `glab auth status` after the user confirms. Don't trust their "done" without verification.

If verification fails:

```
I don't see <platform> credentials yet. Did the login finish in Terminal?
You should have seen a "Logged in as <name>" line at the end.
If you got an error, paste it here and I'll help.
```

Do not loop forever - after two failed verifications, offer to skip Phase F.

**Pause boundary.** Confirm before Phase D.

---

## Phase D - Skills pack (step 11)

> *Why this matters: this installs the jr-rails-* skills (new, classic, phlex, second-opinion, review) into your global skills directory so Claude Code can use them in any project.*

```bash
npx skills add julianrubisch/skills -g -y
```

Verify:

```bash
ls ~/.agents/skills 2>/dev/null | grep '^jr-rails-'
```

Expect at minimum: `jr-rails-new`, `jr-rails-classic`, `jr-rails-phlex`. If any is missing, the install partially failed - re-run the `npx skills add` once. If still incomplete, surface the issue and let the user decide.

Print the discovered skills:

```
✓ Installed jr-rails-* skills:
  - jr-rails-new
  - jr-rails-classic
  - jr-rails-phlex
  - jr-rails-second-opinion
```

**Pause boundary.** Confirm before Phase E.

---

## Phase E - Rails app (steps 12-14)

Branches on the `BRANCH` captured in Phase A.

### Fresh branch (steps 12a-14a)

**Step 12a: Project folder.**

```
Where should I create the project? Default: ~/code
```

Accept the default on empty input. Create the directory if missing (`mkdir -p`). `cd` into it.

**Step 13a: App name.**

```
What should I call the app? (lowercase, no spaces, e.g. my_blog)
```

Validate against `^[a-z][a-z0-9_]*$`. Re-ask if invalid with a one-line reason (`Rails app names start with a lowercase letter and use only lowercase letters, digits, and underscores`).

**Step 14a: Delegate to jr-rails-new.**

Invoke the `jr-rails-new` skill, passing the app name. **Do not interfere** with its 10-question interview. After it returns, capture the app's absolute path as `PROJECT_PATH`.

### Existing branch (steps 12b-14b)

**Step 12b:** Use the `PROJECT_PATH` captured in Phase A. `cd` into it.

**Step 13b: Smoke check.** Verify the directory looks like a Rails app:

```bash
test -f bin/setup && \
test -f Gemfile && \
test -f config/database.yml
```

If any check fails, re-ask for the path (max 2 retries). After 2 failed retries, offer to switch to the fresh-app branch.

**Step 14b:** No app generation. Continue to Phase F.

**Pause boundary.** Confirm before Phase F.

---

## Phase F - Connect to git hosting (steps 15-17)

### Step 15: Check existing remote

```bash
cd "$PROJECT_PATH"
git remote get-url origin 2>/dev/null
```

If exit code 0 and a URL prints: `✓ remote is set to <url>`. **Skip to Phase G.** Never override an existing remote.

### Step 16: Set up remote (if none)

If `PLATFORMS` has both `GitHub` and `GitLab`:

- Ask via `AskUserQuestion`: "Which platform for this project?"
- Options: the selected platforms.

Otherwise use the single selected platform.

Then ask:

- "Create a new repo on <platform>, or push to an existing remote (paste URL)?"

**New repo path:**

GitHub:

```bash
gh repo create <name> --private --source=. --remote=origin --push
```

GitLab:

```bash
glab repo create <name> --private && \
git remote add origin <url-glab-printed> && \
git push -u origin main
```

(Adjust if `glab` changed its create-and-push flags - check `glab repo create --help` once.)

**Existing remote path:**

Validate the URL looks like a git URL (`^https?://.*\.git$` or SSH form, though we steer to HTTPS).

```bash
git remote add origin <url>
git push -u origin main
```

### Step 17: Verify

```bash
git remote -v
```

Report `✓ origin = <url>`.

**Pause boundary.** Confirm before Phase G.

---

## Phase G - Run the app (steps 18-20)

### Step 18: Read project CLAUDE.md

```bash
test -f "$PROJECT_PATH/CLAUDE.md" && cat "$PROJECT_PATH/CLAUDE.md"
```

If present (and the app was created by jr-rails-new, it will be), look for a "How to start the dev server" section. The exact command varies with the worktree / devcontainer choices made in jr-rails-new's interview. Use whatever it specifies.

Default fallback if no project CLAUDE.md or no dev-server section:

```bash
bin/dev
```

### Step 19: Start and wait for ready

Start the dev server in the background. Wait for one of:

- `Listening on http://127.0.0.1:3000`
- `Listening on tcp://0.0.0.0:3000`
- Vite-style `dev server running at http://localhost:3000`

Don't poll forever. Timeout at 90 seconds. If no ready message:

```
Dev server didn't start cleanly within 90s. Last 30 lines of output:
<tail of log>
```

### Step 20: Open browser

```bash
open http://localhost:3000
```

Print:

```
🎉 Your Rails app is running at http://localhost:3000
   Server log: <path>
   Stop with Ctrl-C in this terminal, or kill <PID>.
```

---

## Final artifact

Write `$PROJECT_PATH/SETUP_SUMMARY.md` using `references/setup-summary-template.md`. Fill in:

- macOS version, Ruby version, Node version, Homebrew version, mise version
- gh/glab installation + auth status
- Git remote URL
- Skills installed
- Server command and URL
- Date of run (today's date)

Print the final summary to the user:

```
✅ Setup complete. Wrote SETUP_SUMMARY.md.

Next time you want a new Rails app, just run /jr-rails-new directly - the
machine is ready. Use /jr-rails-classic for coding style guidance and
/jr-rails-review for code review.
```

---

## Error handling

Plain-language template for any failure:

```
Step <N> hit a snag.

What went wrong: <one sentence in plain English>
What I tried: <command>
What the system said: <last 5 lines of output, indented>

Suggested fix: <one or two sentences>

Want me to retry, skip this step, or stop here?
```

Never paste an unannotated stack trace. Translate first, then show evidence.

---

## Re-running on a configured Mac

If every Phase B/C/D check reports `✓ already installed` / `✓ already authenticated` and the user is on the `fresh` branch, you'll loop back to jr-rails-new for a second app. That's fine, but warn explicitly at Phase A:

```
This machine looks bootstrapped already. The fresh-app path will run
/jr-rails-new for a second project. The existing-app path lets you point
me at the one you already have. Which do you want?
```

If they pick fresh, proceed normally. If existing, take the existing branch and skip straight to Phase E step 12b.
