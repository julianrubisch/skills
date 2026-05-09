# CLI Invocations

Canonical review-style invocations per agentic CLI. Flags drift between releases; verify with `<cli> --help` on first run for each CLI.

To add a new CLI: append a section below, then update the detection loop in `SKILL.md` and `guide.md`.

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

Opencode has a built-in `/review` slash command exposed via `--command`.

```bash
opencode run \
  --dir "$REPO_ROOT" \
  --command review \
  "<branch-or-arg>"
```

For uncommitted changes, omit the positional arg or pass `HEAD`. The `/review` command interprets the argument as either a branch, a commit SHA, or a PR number.

**Permissions:** the agent runs in the project; on first run you may be prompted to allow tool use. Use `--dangerously-skip-permissions` only when the run is fully trusted.

**Output:** formatted Markdown to stdout. Add `--format json` for parseable output.

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
  opencode run --dir "$PWD" --command review "$BRANCH" > /tmp/review.opencode.md &
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
