# Phase C Step 10: Auth Handoff

The hardest UX moment in the bootstrap. `gh` and `glab` need to write credentials to the host shell's keychain/config. Running them inside the Claude Code Bash tool is unreliable: the browser-based OAuth flow needs a TTY, the credential helpers may target the wrong context, and tool-output buffering can swallow the verification token paste.

**Rule: auth always runs in a separate Terminal tab/window opened by the user.**

The user is already in Terminal (that's where Claude Code is running). Don't tell them to "open Terminal.app" - tell them to open a new tab or window.

---

## Detecting current auth state

Before instructing anything, check:

```bash
gh auth status 2>&1
```

Exit code 0 and output contains `Logged in to github.com as <user>`: skip, report `✓ already authenticated as <user>`.

```bash
glab auth status 2>&1
```

Exit code 0 and output contains `Logged in to <host> as <user>`: skip.

---

## Handoff script (paste this to the user, verbatim)

When auth is needed for `gh`:

```
⚠️  This step happens in a separate Terminal tab, not here.

1. Open a new Terminal tab (Cmd-T) or window (Cmd-N) - you're already
   in Terminal, you just need a fresh shell that isn't running me.
2. In that new tab, paste:

       gh auth login

3. Pick these options when prompted:
   - "What account do you want to log into?"  →  GitHub.com
   - "What is your preferred protocol for Git operations?"  →  HTTPS
   - "Authenticate Git with your GitHub credentials?"  →  Yes
   - "How would you like to authenticate GitHub CLI?"  →  Login with a web browser
4. A browser tab opens. Copy the one-time code shown in your Terminal
   tab, paste it into the browser, click Authorize.
5. Back in the Terminal tab, you should see "✓ Logged in as <username>".

Tell me "done" when that line appears.
```

Same shape for `glab` - replace command with `glab auth login`. Host prompt is `gitlab.com` (or self-hosted instance if the user is on one).

If both platforms were selected, walk through them sequentially. Don't list both auth commands at once - one at a time, confirm each before moving to the next.

---

## Verification (mandatory after "done")

Never trust the user's "done" word. Re-run the status check:

```bash
gh auth status 2>&1
glab auth status 2>&1
```

If still unauthenticated:

```
I don't see <platform> credentials yet. A few things to check:

• Did you run "gh auth login" in a NEW Terminal tab (not this one)?
• Did you see "✓ Logged in as <name>" in that tab?
• Are you using the same Mac user account in both tabs? (You should
  be - new tabs inherit your user.)

If you saw an error in that tab, paste it here and I'll help.
```

Allow up to **2 verification retries**. After that, offer:

- Skip Phase F entirely (no remote push). The project still works locally.
- Stop here. The user can run `gh auth login` later and re-trigger Phase F manually.

Do not loop indefinitely.

---

## Why we do it this way (for the implementer's reference)

- `gh auth login --with-token < file` would work non-interactively, but requires the user to generate a PAT first - more friction for a non-technical user than the browser flow.
- Running `gh auth login` via the Bash tool with `run_in_background: true` would let the browser flow happen, but the credential write may target the Claude Code session's keychain context rather than the user's host login shell. Result: subsequent steps see "not authenticated" even after a successful OAuth dance.
- The separate-tab handoff is uglier UX but always correct. Worth the friction.
