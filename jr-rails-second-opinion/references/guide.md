# Rails Second Opinion Guide (Local Agentic CLI)

Wrap a locally-installed agentic CLI in a structured review loop. The CLI is a signal generator. The mediator (you) owns synthesis and decisions.

---

## Severity Rubric

| Severity | Definition | Gate impact |
|----------|------------|-------------|
| **High** | Blocks shipping. Incorrect, unsafe, or fundamentally broken (security hole, data loss, broken migration, broken test). | Must fix before close. |
| **Medium** | Should fix. Quality or maintainability issue but not blocking (N+1, missing index, leaky abstraction, weak test). | Fix or explicitly accept risk. |
| **Low** | Nice to have. Polish, naming, minor refactors. | Fix if time permits. |

Both the implementer self-review and the CLI external review label findings with H/M/L. The mediator uses these to enforce gates.

---

## Roles

- **Implementer**: the agent doing the Rails work. Also self-reviews. Proposes synthesis; does not approve design decisions.
- **Mediator**: the human (you). Approves synthesis, reconciles disagreements, decides what changes. All design decisions require explicit approval.
- **Expert**: the chosen agentic CLI (claude, codex, opencode, gemini, aider, etc.). Independent external lens. When this skill is invoked from a non-Claude harness (e.g. running inside codex), `claude` itself becomes a valid expert target.

Critical: the implementer must STOP and escalate to the mediator when design decisions arise. The implementer writes synthesis proposals; the mediator owns final decisions.

---

## Modes

### Two-CLI mode (default)
One implementer (self-review) + one CLI (external review). Use when iterating quickly or when one dominant concern (correctness, security) drives the review.

### Multi-CLI mode (`--multi`)
Implementer + 2+ CLIs running in parallel. Use for high-stakes artifacts where diverse model perspectives are valuable. Reconciliation cost is higher; budget accordingly.

---

## Phase 0: CLI Discovery (Required)

Before any review, probe `$PATH` for installed agentic CLIs:

```bash
for cmd in claude codex opencode gemini aider mods cursor-agent llm sgpt goose; do
  path=$(command -v "$cmd" 2>/dev/null) || continue
  echo "$cmd -> $path"
done
```

Then call `AskUserQuestion`:

- **0 detected**: STOP. Tell the mediator no agentic CLI is on `$PATH` and suggest installing one (`brew install codex`, `npm i -g opencode-ai`, `pip install aider-chat`, etc.).
- **1 detected**: confirm it as the choice (single-option `AskUserQuestion`).
- **2+ detected**: list as options. Add an "all (multi mode)" option iff `--multi` was passed.

Record the chosen CLI in the working log under `## Reviewers`.

For invocation patterns per CLI, load `cli-invocations.md`.

---

## Phase 1: Setup (Gate)

Define the review contract before any review happens.

### Inputs
- **Artifact**: uncommitted changes (default), branch diff, or PR diff.
- **Scope**: what is in/out (which paths, which dimensions).
- **Quality bar**: what "S-tier" means here (production-ready? prototype? PR for a critical path?).
- **Time budget**: rounds (typically 1-2 for a CLI second opinion).

### Reviewer roster
- Two-CLI mode: one CLI from Phase 0. That's it.
- Multi-CLI mode: 2-4 CLIs from Phase 0.

### Create the working log
Copy `second-opinion-log-template.md` to the working dir as `second-opinion.md`. This is a session artifact; it will be deleted after attestation.

Add `second-opinion.md` to `.gitignore` (project or global) to prevent accidental commits.

**Gate to proceed:**
- Artifact and scope stated in the log.
- Chosen CLI(s) recorded under `## Reviewers`.
- `second-opinion.md` exists.

---

## Phase 2: Round Loop

Each round has four steps: Self-Review → External Review → Reconcile → Synthesize.

### Step 0: Self-Review (Implementer)
Before invoking the CLI, the implementer reviews their own work with a fresh, critical eye. Produce H/M/L findings, no softball.

The Rails dimensions to scan against:

- **Architecture**: skinny controllers, rich domain models, no service objects, callbacks scored, concerns over-applied.
- **Quality**: idiomatic Ruby, naming, dead code, redundant abstractions, anti-patterns from Ruby Science.
- **Performance**: N+1, missing indexes, eager-load opportunities, query object candidates, cache strategy.
- **Testing**: coverage of new code, integration vs unit balance, fixture/factory hygiene, system test smell.
- **Security**: strong params, mass assignment, escape sites, Brakeman class warnings, auth boundaries.

### Step A: External Review (CLI)

Build the brief and invoke the CLI. Brief template (insert into the CLI invocation; see `cli-invocations.md` for per-CLI placement):

```
ROLE: senior Rails reviewer. You are doing a code review, not writing code.

ARTIFACT: <branch | uncommitted | PR #N> on this Rails repository.

SCOPE:
- Files touched: see the diff
- Out of scope: unrelated files, generated files (db/schema.rb, etc.), vendor/, node_modules/

QUALITY BAR: <prototype | merge-ready | production-critical>

CONSTRAINTS:
- Severity-label every finding: H / M / L. No findings without a label.
- Be specific: cite file:line, not vibes.
- No service objects. Do not recommend extracting logic into *Service, *Manager,
  *Handler, *Processor, *Creator, or single-method .call POROs. When extraction
  is needed, prefer: domain model, form object, query object, concern, DCI
  context, callback extraction.
- Avoid generic advice the diff already implies.
- Do not edit files. This is a review, not a refactor.

DIMENSIONS to consider (Rails-flavored):
- Architecture (skinny controllers, rich models, callback design)
- Quality (Ruby idiom, naming, anti-patterns)
- Performance (N+1, indexes, eager loading)
- Testing (coverage, integration/unit balance)
- Security (strong params, mass assignment, Brakeman class issues)

OUTPUT FORMAT:
1. Findings (ordered by severity, H first), each labeled H/M/L with file:line citation
2. Risks / missing considerations
3. Suggested verification (tests to add, things to grep for)
4. Confidence: low / medium / high
```

Per-CLI invocation: see `cli-invocations.md`. Capture stdout into the working log under `### External Review (Round N)`.

### Step B: Reconcile

Compare self-review and CLI findings. Produce a reconciliation table:

| Issue | Self-Review | CLI | Agreement |
|-------|-------------|-----|-----------|
| ... | Found H | Found H | Agree |
| ... | Missed | Found M | Add to actions |
| ... | Found L | Missed | Keep, drop CLI input |
| ... | Found H | Disagrees | Steelman both, escalate |

Where self-review and CLI disagree, steelman both perspectives. If the disagreement is a design tradeoff, escalate per the trigger list below.

### Step C: Synthesize (Implementer Proposes, Mediator Approves)

```
### Round N Synthesis

**Consensus:**
- ...

**Disagreements:**
- [Self-Review] vs [CLI]: [topic]
  - Decision: [what we do and why]

**Actions:**
- [ ] Action 1
- [ ] Action 2

**Decision points:** none this round.
(or full DECISION POINT template if triggers fired)

**Open Questions:**
- ...

**Gate Status:**
- Open high-severity items? [yes/no, must be no to close]
- Open medium items accepted? [yes/no/N/A]
- All actions addressed? [yes/no]
- Ready to close? [yes/no]

**Mediator Approval:** pending
```

Wait for mediator approval before acting. When given, replace `pending` with `approved by NAME`.

### Step D: Implement (Between Rounds)

After approval, the implementer addresses agreed actions. The next round opens with `### Changes (Round N)` documenting what was done, then re-runs the loop on the updated artifact.

**STOP and escalate** if any action involves:

| Trigger | Action |
|---------|--------|
| Design tradeoff | Present options A/B with pros/cons via `AskUserQuestion` |
| "By design" response from CLI | Don't dismiss; escalate to mediator |
| Scope change | Confirm before proceeding |
| Repeated issue (2+ rounds) | May indicate deeper problem; flag it |
| Accepting a limitation | Mediator decides, not implementer |
| Architectural choice | Affects overall structure; escalate |

### Step E: Round 2+ Ping-Back

For Round 2 and later, re-run the CLI but ask for **deltas only**:

```
This is Round N. Here is what changed since the previous round:

<diff or summary of changes>

Synthesis from previous round:
<paste synthesis>

Respond with:
- Critical misses (H/M/L labeled)
- Disagreement with previous decisions
- Any new issues (H/M/L labeled)

Do not repeat findings from previous rounds. Use H/M/L severity labels.
```

---

## Phase 3: Convergence Gate

Stop when one of:

- Two consecutive rounds with no open H AND all M either fixed or explicitly accepted by mediator.
- No open H, all actions addressed, CLI has no new deltas, all M either fixed or accepted.
- Time budget reached and risk explicitly accepted by mediator (not by the implementer). Document which H/M items remain and why.

### Required Attestation

The mediator must include in the final synthesis:

```
**Attestation:** I confirm this log reflects genuine review work, not template filling. - [NAME]
```

This is the real gate. Eval checks below are smoke tests.

### Cleanup

After attestation:

1. Delete `second-opinion.md`. The improved artifact is the deliverable, not the log.
2. Confirm `second-opinion.md` is in `.gitignore` (project or `~/.gitignore_global`).

---

## Eval Checks

Smoke tests for the working log. Pass/fail is manual; counts print for human verification.

| # | Check | Command | Pass |
|---|-------|---------|------|
| 1 | Log exists | `test -f second-opinion.md` | File exists |
| 2 | Reviewer recorded | `grep -q "^## Reviewers" second-opinion.md` | Exit 0 |
| 3 | CLI recorded under reviewers | `grep -qE "(codex|opencode|gemini|aider|mods|cursor-agent|llm|goose)" second-opinion.md` | Exit 0 |
| 4 | Each round has synthesis | `grep -c "^### Round .* Synthesis" second-opinion.md` | Count = rounds |
| 5 | Synthesis sections present | `grep -E "^\*\*(Consensus\|Disagreements\|Actions\|Gate Status):" second-opinion.md` | All 4 per round |
| 6 | H/M/L findings captured | `grep -qE "(^|[[:space:]-])([HML]:\|High:\|Medium:\|Low:)" second-opinion.md` | Exit 0 |
| 7 | Self-Review per round | `grep -c "^### Self-Review" second-opinion.md` | Count = rounds |
| 8 | External Review per round | `grep -c "^### External Review" second-opinion.md` | Count = rounds |
| 9 | Reconciliation per round | `grep -c "^### Reconciliation" second-opinion.md` | Count = rounds |
| 10 | Mediator approval per round | `grep -cE "^\*\*Mediator Approval:\*\* approved by [^[]" second-opinion.md` | Count = rounds |
| 11 | Changes logged (rounds > 1) | `grep -c "^### Changes" second-opinion.md` | Count = rounds - 1 |
| 12 | Final synthesis | `grep -q "^## Final Synthesis" second-opinion.md` | Exit 0 |
| 13 | Attestation | `grep -qE "^\*\*Attestation:\*\*.+- [^[]" second-opinion.md` | Exit 0 |

Quick check script:

```bash
LOG=second-opinion.md
echo "1) Log exists:"; test -f $LOG && echo PASS || echo FAIL
echo "2) Reviewer recorded:"; grep -q "^## Reviewers" $LOG && echo PASS || echo FAIL
echo "3) CLI named:"; grep -qE "(codex|opencode|gemini|aider|mods|cursor-agent|llm|goose)" $LOG && echo PASS || echo FAIL
rounds=$(grep -c "^## Round" $LOG || echo 0)
echo "4) Rounds detected: $rounds"
echo "5) Synthesis sections:"; grep -E "^\*\*(Consensus|Disagreements|Actions|Gate Status):" $LOG | wc -l
echo "6) H/M/L findings:"; grep -qE "(^|[[:space:]-])([HML]:|High:|Medium:|Low:)" $LOG && echo PASS || echo FAIL
echo "7) Self-Review per round:"; grep -c "^### Self-Review" $LOG
echo "8) External Review per round:"; grep -c "^### External Review" $LOG
echo "9) Reconciliation per round:"; grep -c "^### Reconciliation" $LOG
echo "10) Mediator approval:"; grep -cE "^\*\*Mediator Approval:\*\* approved by [^[]" $LOG
echo "11) Changes (rounds>1):"; grep -c "^### Changes" $LOG
echo "12) Final synthesis:"; grep -q "^## Final Synthesis" $LOG && echo PASS || echo FAIL
echo "13) Attestation:"; grep -qE "^\*\*Attestation:\*\*.+- [^[]" $LOG && echo PASS || echo FAIL
```

---

## Failure Modes

| Symptom | Cause | Fix |
|---|---|---|
| CLI returns generic Rails advice | Brief was too thin | Pass file:line scope and dimension list explicitly |
| CLI suggests service objects | Hard rule not in brief | Always include the "no service objects" line |
| CLI prints nothing | Auth prompt or rate limit | Run the CLI interactively once to clear |
| Findings without severity labels | Format constraint not enforced | Reject and re-run with explicit H/M/L instruction |
| Reviewer keeps repeating | No delta-only constraint in Round 2+ | Use the ping-back template |
| Convergence stalls | Disagreements not mediated | Force explicit decisions in synthesis |
| Implementer accepts CLI verdict without thinking | Skipped self-review | Self-review is non-negotiable; do not skip |
| CLI tries to edit files | Missing `--no-write` style flag | Add the no-write flag for that CLI; never act on file mutations |
| Diff includes secrets | `.env` / credentials in scope | Scrub before piping; let Brakeman flag any new secret-looking files |
| Eval checks pass on placeholder text | Pattern matched template | Use `[^[]` style negation in checks |

---

## Working Log

The working log lives at `second-opinion.md` in the working directory. Copy `second-opinion-log-template.md` as a starting point.
