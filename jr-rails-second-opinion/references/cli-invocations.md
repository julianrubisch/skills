# CLI Invocations

Canonical review-style invocations per agentic CLI. Flags drift between releases; verify with `<cli> --help` on first run for each CLI.

To add a new CLI: append a section below, then update the detection loop in `SKILL.md` and `guide.md`.

---

## claude (Anthropic Claude Code CLI)

Use when this skill is invoked from a non-Claude harness (codex, opencode, etc.) and you want Claude as the external reviewer. Run as a one-shot, headless prompt.

```bash
# Pipe diff into a one-shot Claude run
git diff "$BASE"..."$HEAD" | claude -p "$(cat <<EOF
$BRIEF

Review the diff on stdin. Output H/M/L findings only. Do not edit files.
EOF
)"
```

Flags:
- `-p` / `--print` runs non-interactively and prints the response to stdout.
- Add `--model claude-sonnet-4-6` (or another model id) to pin the reviewing model — keep it different from whatever ran the work to maximize the value of the second opinion.
- Add `--permission-mode plan` (or `--allowed-tools` empty) to prevent the reviewing instance from writing files. The brief instructs no edits; the flag is belt-and-suspenders.

**Auth:** `claude` uses your existing Anthropic account / API key. No extra setup if Claude Code is already installed.

**Output:** Markdown to stdout.

**Self-review caveat:** if Claude is also the implementer, running Claude as the external reviewer collapses the value of the loop. Prefer a different vendor when possible; reserve `claude` as the reviewer for runs originating outside Claude Code.

---

## codex (OpenAI Codex CLI)

Codex has a built-in `review` subcommand. Use it directly.

```bash
# Review uncommitted changes (staged + unstaged + untracked)
codex review --uncommitted "$BRIEF"

# Review a branch diff against a base
codex review --base main "$BRIEF"

# Review a single commit
codex review --commit <sha> "$BRIEF"
```

`$BRIEF` is the brief constructed per `guide.md`. The `[PROMPT]` argument can be `-` to read from stdin (useful for very long briefs).

**Output:** stdout, structured Markdown.

**Auth:** `codex login` (one-time). Verify with `codex --help`.

---

## opencode (sst/opencode)

There is no `review` subcommand in current builds. Pass the brief as the message to `opencode run`.
Verify against `opencode run --help` — this CLI revs flags often.

```bash
# One-shot review, pinned model, backgrounded with pollable progress
cd "$WORKTREE" && OPENCODE_CONFIG="$SCRATCH/oc-config.json" nohup opencode run \
  --print-logs \
  -m opencode-go/kimi-k2.7-code \
  --title "second-opinion-<issue>" \
  "$BRIEF" < /dev/null > "$SCRATCH/review.md" 2> "$SCRATCH/review.err" &
```

**Always close stdin (`< /dev/null`) on a backgrounded run.** See "a backgrounded run waits on
stdin" below; it is the most common hang and the cheapest to rule out.

Flags:
- `run [message..]` is the one-shot mode; the brief goes in as the positional message.
- `-m` / `--model` takes `provider/model`. **The full provider path is required** for OpenRouter:
  `openrouter/kimi-k3` does not resolve, `openrouter/moonshotai/kimi-k3` does. First-party providers
  are flat: `opencode-go/kimi-k2.7-code`. Confirm with `opencode models | grep <name>` first — a bad
  id fails slowly and silently, not loudly.
- `--print-logs` sends structured logs to stderr while the answer still goes to stdout. **Use it for
  every backgrounded run.** Without it a long run and a hung one look identical — a 0-byte output file
  either way — and you cannot tell which you have until the timeout. With it, `tail` the `.err` file
  and watch `message=loop step=N` advance.

**Provider choice.** Prefer a first-party provider (`opencode/…`, `opencode-go/…`) over routing the
same model through `openrouter/…`. OpenRouter has been the source of multi-hour degradations that
present as an unexplained hang, and swapping the provider path is a one-flag test that rules it out.
Same model, different route.
- `--format json` emits raw JSON events instead of the formatted stream. Note this is `--format json`,
  **not** `--json`; there is no `--json` flag.
- `--agent` selects an agent. There is no built-in read-only `plan` agent (`opencode agent list`
  shows `build`, `compaction`, `explore`), so **no flag guarantees no-write**.
- `--pure` skips external plugins. Worth adding if `~/.config/opencode/plugins/*` logs load errors.

**No-write enforcement:** since no flag guarantees it, state "DO NOT EDIT ANY FILES" in the brief
*and* verify afterwards with `git status --short`. Commit before running so any mutation is visible
and revertible with `git checkout .`.

**Auth:** `opencode providers list` shows configured credentials.

### Known failure: a backgrounded run waits on stdin

**Check this first.** `opencode run` reads piped stdin and appends it to the message, so it waits for
EOF before it starts. A foreground call from a harness usually gets `/dev/null` as stdin and works; a
backgrounded one (Claude Code's `run_in_background`, `&`, `nohup`) can inherit a pipe that never
closes, and the run sits at `message=init` forever: no `message=loop`, no session, 0-byte output.
Same signature as tool denial below, which is why it gets misdiagnosed.

Discriminator: the identical command succeeds in the foreground (under a `timeout`) but hangs when
backgrounded, and toolless and tool-using smoke tests both pass in the foreground. Fix: redirect
stdin, `< /dev/null`, on every backgrounded run. Observed 2026-09: two long-brief runs hung 30+
minutes each at `init`; `--title`, the brief and the provider were ruled out one at a time; adding
`< /dev/null` got the next run to the loop within seconds.

### Known failure: provider quota errors look like a hang without `--print-logs`

A plan cap (`AI_APICallError: Go usage limit exceeded`) or a disabled model on a pay-per-use route
(`Upstream request failed: Model access is disabled`) fails the stream at step 1, and the run can
then sit without exiting or printing anything. Only the `.err` log shows it:
`grep -o 'error.error=.*' review.err`. Check this before debugging anything else in a hung run; the
fix is a different route for the same model (`opencode-go/…` → `opencode/…` → `openrouter/…`) or
waiting for the reset, and it is a billing decision for the mediator.

### Known failure: silent hang because the config denies the reviewer's tools

**Check this after stdin.** `opencode run` produces zero output and never exits when the agent's tools are
denied by config. A review needs to read the diff and the files around it; with no usable tool it
stalls instead of failing. The log stops at `message=init`, no session is recorded, and the redirect
target stays 0 bytes — indistinguishable from a crash.

```bash
grep -A8 '"permission"' ~/.config/opencode/opencode.json
```

A lean-ctx-style setup denies the native tools on purpose, in favour of `ctx_*` MCP equivalents:

```json
"permission": { "bash": "deny", "glob": "deny", "grep": "deny", "read": "deny" }
```

**`--dangerously-skip-permissions` does not help** — it auto-approves what is not *explicitly* denied,
and these are explicitly denied.

**Fix: override via `OPENCODE_CONFIG` for the run.** Never edit the user's config. **`OPENCODE_CONFIG`
REPLACES the global config, it does not merge with it** — so build the override by merging your
permission block into a *copy* of the user's config, never as a standalone file (see the next
section for what a standalone file costs you):

```bash
jq '. + {permission: {bash: "allow", glob: "allow", grep: "allow", read: "allow",
                      edit: "deny", write: "deny", patch: "deny"}}' \
  ~/.config/opencode/opencode.json > /tmp/oc-review-config.json
export OPENCODE_CONFIG=/tmp/oc-review-config.json
opencode run --print-logs -m <model> "$BRIEF" > review.md 2> review.err
```

Denying `edit`/`write`/`patch` in that same override is how you get the no-write guarantee the CLI
otherwise lacks. Merging is what keeps the user's `provider`, `mcp` and `plugin` blocks alive, so
MCP servers still resolve and the agent can use `ctx_*` tools where they exist.

Check which file actually holds the config before merging (`ls ~/.config/opencode/`). Both
`opencode.json` and `opencode.jsonc` are valid names, opencode logs the path it *tries*, and
grepping the wrong one returns a confident, wrong "no permission block".

### Known failure: a standalone OPENCODE_CONFIG silently drops the provider config

Same 0-byte, hangs-at-`init` signature as tool denial, different cause and different fix. Writing the
override as a fresh `{"$schema": ..., "permission": {...}}` file replaces the user's config wholesale;
`provider` (and `mcp`, `plugin`) go with it, and a run that needs a tool stalls at init instead of
failing. Observed on a machine whose config had **no** permission block at all — so the override was
not even needed, and adding it was what broke the run.

**Discriminator, cheaper than any other test:** a *toolless* prompt still succeeds under a replacing
config, while a tool-using one hangs. Under genuine tool denial, both behave the same way (the
toolless one passes because it needs nothing). So run both:

```bash
# A: toolless + override      -> succeeds under replacement, succeeds under denial
OPENCODE_CONFIG=$CFG opencode run -m <model> "Reply with exactly: PLUMBING OK"
# B: tool-using, NO override  -> succeeds under replacement, hangs under denial
opencode run -m <model> "Run 'git rev-parse --abbrev-ref HEAD' and reply with exactly: BRANCH=<name>"
```

A passing and B passing means the config you wrote is the problem, not the CLI, not the model, and
not the project. Merge instead of replacing and re-test B with the override.

**Smoke-test with a prompt that USES a tool.** This is the trap that cost a session: a prompt like
"Reply with exactly: PLUMBING OK" needs no tools, so it succeeds under a config that makes review
impossible. It proves the CLI can talk, not that it can work.

```bash
opencode run -m <model> "Run 'git rev-parse --abbrev-ref HEAD' and reply with exactly: BRANCH=<name>"
```

Red herrings ruled out by controlled test, in the order they seduce you — do not spend a session on
them again:

- **Flags.** `--format json`, `--pure`, `--dangerously-skip-permissions`: none changes the behaviour.
- **A missing TTY.** The same command hangs identically for a human in a real terminal.
- **`.mcp.json` / project MCP servers.** Moving it aside *appears* to fix the hang, because by the
  time you try it the tool-less prompt you are testing with has started passing anyway. Re-tested
  with `.mcp.json` in place: fine. Editing a tracked file to "fix" this is pure damage.
- **Cold-start indexing.** Plausible, and wrong: a warm project with denied tools still hangs on any
  prompt that needs to read something.
- **`--print-logs` as a cure.** It is a *diagnostic*, not a fix. A run that appears to start working
  once you add it was going to work anyway; what changed is that you can now see it working. Adding
  it is still right — just do not record it as having solved anything.
- **Switching provider to escape a hang.** Moving off `openrouter/…` genuinely fixes a provider
  *degradation* (slow or failing upstream) and genuinely does nothing for either failure above. Try
  it once, early, because it is cheap; do not keep re-trying it against a symptom it cannot touch.
- **Re-running with a longer timeout.** For the reasoning-burn failure this just buys the same
  outcome more slowly. Fix the brief or recover the session.

**Beware harness timeouts.** A foreground call migrated to background at a timeout can kill the child
*and skip its `trap`*, leaving temporary edits unreverted. Launch long runs in the background from
the start, and verify the worktree with `git status --short` afterwards.

### Known failure: a complete run that outputs nothing

**A different failure from the one above, with the same 0-byte symptom.** Distinguish them by the
exit and the log, not by the output file:

| | Tool denial (above) | Reasoning burn (here) |
|---|---|---|
| Exit | never — hangs until killed | 0, cleanly |
| Log tail | stops at `message=init` | `exiting loop`, `disposing instance` |
| Steps | none | several `message=loop step=N` |
| Duration | unbounded | a full, plausible run |

The cause is the model spending its entire final turn inside a reasoning block and never emitting a
text part. Observed: 131,000 characters of reasoning, much of it degenerating into repetition
("Potential issue with X. Good." over and over), and no answer. The CLI is working perfectly; there
is simply nothing on stdout to print.

**Prevent it in the brief.** State plainly that the findings must be the final message, that the
model should think briefly and then write, and that a partial answer beats a perfect unwritten one.
`guide.md`'s brief template carries this constraint — do not drop it when adapting the brief.

**Recover the run rather than repeating it.** The analysis is in the session store, and re-running
costs another 5–20 minutes for a result that may fail the same way. Since v1.17 opencode keeps
sessions in SQLite, not the old `storage/*/` directories:

```bash
DB=~/.local/share/opencode/opencode.db
SESS=$(grep -oE 'ses_[A-Za-z0-9]+' "$SCRATCH/review.err" | tail -1)

# What the final turn actually produced
sqlite3 "$DB" "SELECT rowid, json_extract(data,'\$.type'), length(data)
               FROM part WHERE session_id='$SESS' ORDER BY rowid;" | tail -20

# Pull the reasoning out (substitute the rowid of the large trailing reasoning part)
sqlite3 "$DB" "SELECT json_extract(data,'\$.text') FROM part WHERE rowid=<ROWID>;" \
  > "$SCRATCH/review-reasoning.md"
```

Then mine it rather than reading it end to end — it is unstructured and mostly noise:

```bash
grep -niE "severity|this is (a bug|wrong|incorrect)|real bug|should be fixed" "$SCRATCH/review-reasoning.md"
```

**Treat what you recover as findings, not as a review.** It never went through whatever
self-editing produces the final answer, so it contains abandoned lines of thought and conclusions
the model later talked itself out of. Verify every claim against the code before it reaches the
reconciliation table — which is the standing rule anyway.

---

## gemini (Google Gemini CLI)

No built-in review subcommand. Pipe the diff into a prompt.

```bash
git diff "$BASE"..."$HEAD" | gemini -p "$(cat <<EOF
$BRIEF

Diff follows on stdin.
EOF
)"
```

`-p` is one-shot. Drop it for an interactive session.

---

## aider (paul-gauthier/aider)

Aider is a coding assistant; review-style is a one-shot message.

```bash
aider --message "$(cat <<EOF
$BRIEF

Review the diff between $BASE and $HEAD. Do not edit files. Output H/M/L findings only.
EOF
)" --no-auto-commits --read .
```

`--no-auto-commits` is critical: aider will otherwise try to apply edits. We want findings, not changes.

---

## mods (charmbracelet/mods)

Lightweight stdin-based prompt runner. Best for quick review with a smaller model.

```bash
git diff "$BASE"..."$HEAD" | mods "$BRIEF

Review the diff above. Output H/M/L findings only."
```

---

## cursor-agent (if installed)

Cursor's headless agent. Verify the binary exists and check `--help` for the current invocation.

```bash
cursor-agent --message "$BRIEF" --read . --no-write
```

---

## llm (simonw/llm)

Pure prompt runner; useful when you want to choose a specific model.

```bash
git diff "$BASE"..."$HEAD" | llm -m gpt-4o "$BRIEF

Review the diff above. Output H/M/L findings only."
```

---

## goose (block/goose)

Block's open-source agent.

```bash
goose run --instructions "$BRIEF

Review the staged diff. Output H/M/L findings only."
```

---

## Multi-CLI Mode

When `--multi` is passed, run two or more CLIs in parallel and reconcile:

```bash
{
  codex review --base main "$BRIEF" > /tmp/review.codex.md &

  wait
}
```

Reconciliation is the implementer's job: open both outputs, dedupe, mark consensus vs disagreement in the synthesis. See `guide.md` Phase 2.

---

## Adding a New CLI

1. Find the canonical review invocation. Prefer a built-in review subcommand if one exists. Fall back to piping `git diff` into a prompt.
2. Add a section above following the same template (heading, code block, notes on auth/output/quirks).
3. Add the binary name to the detection loop in `SKILL.md` and `guide.md`.
4. Note any flags that prevent file writes (`--no-write`, `--no-auto-commits`, etc.). Review must not mutate the working tree.

---

## Caveats

- **Flag drift.** Every CLI revs invocation flags between releases. Re-verify with `--help` when a CLI starts behaving oddly.
- **Auth failures surface as silent stdouts.** If a CLI prints nothing, run it manually first to clear an auth prompt.

- **Token costs.** Some CLIs default to expensive models. Pin a cheaper model when iterating (`-m`, `--model`, `model="..."` in TOML, etc.).
- **Don't pipe secrets.** Always exclude `.env`, `config/credentials/*`, and any file Brakeman flags before piping a diff to a remote model.
- **PATH is not what you think.** Claude Code's Bash tool runs in a non-login shell. CLIs installed via fish-only PATH config, asdf/mise/volta shims, or `npm`/`bun`/`pnpm` global bin dirs may be invisible. Phase 0 in `guide.md` resolves the login-shell PATH; reuse the resulting `$SEARCH_PATH` (or capture the absolute binary path) when invoking the CLI in later phases. If detection returns 0 hits but you know a CLI is installed, ask the mediator for an explicit path before suggesting an install.
