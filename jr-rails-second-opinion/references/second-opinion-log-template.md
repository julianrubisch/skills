# Second Opinion Log

Copy this file to the working directory as `second-opinion.md` when starting a session. Add `second-opinion.md` to project `.gitignore` or `~/.gitignore_global` so it does not get committed.

After final attestation: delete this file. The improved artifact is the deliverable, not the log.

---

## Artifact

- Description: <what is being reviewed: branch name, PR #, "uncommitted">
- Scope: <paths in scope, paths excluded>
- Quality bar: <prototype | merge-ready | production-critical>
- Time budget: <N rounds | minutes>

## Reviewers

- Implementer: self-review
- External CLI: <codex | opencode | gemini | aider | mods | cursor-agent | llm | goose>
- Multi-CLI mode: <yes | no>

---

## Round 1

### Self-Review (Implementer)

H:
- ...

M:
- ...

L:
- ...

### External Review (<CLI name>)

Command run:
```bash
<paste exact invocation>
```

Output:

H:
- ...

M:
- ...

L:
- ...

### Reconciliation

| Issue | Self-Review | CLI | Agreement |
|-------|-------------|-----|-----------|
| ... | Found H | Found H | Agree |
| ... | Missed | Found M | Add to actions |
| ... | Found L | Missed | Keep, drop CLI input |

### Round 1 Synthesis

**Consensus:**
- ...

**Disagreements:**
- ...

**Actions:**
- [ ] ...

**Decision points:** none this round.
(or full DECISION POINT template)

**Open Questions:**
- ...

**Gate Status:**
- Open high-severity items? [yes/no]
- Open medium items accepted? [yes/no/N/A]
- All actions addressed? [yes/no]
- Ready to close? [yes/no]

**Mediator Approval:** pending

---

## Round 2

### Changes (Round 1)

- ...

### Self-Review (Implementer)

H:
- ...

M:
- ...

L:
- ...

### External Review (<CLI name>)

Command run:
```bash
<paste exact invocation, with delta-only ping-back prompt>
```

Output:

H:
- ...

M:
- ...

L:
- ...

### Reconciliation

| Issue | Self-Review | CLI | Agreement |
|-------|-------------|-----|-----------|

### Round 2 Synthesis

**Consensus:**
- ...

**Disagreements:**
- ...

**Actions:**
- [ ] ...

**Decision points:** none this round.

**Gate Status:**
- Open high-severity items? [yes/no]
- Open medium items accepted? [yes/no/N/A]
- All actions addressed? [yes/no]
- Ready to close? [yes/no]

**Mediator Approval:** pending

---

## Final Synthesis

- Decision: ...
- Risks accepted: ...
- Next steps: ...

**Attestation:** I confirm this log reflects genuine review work, not template filling. - [NAME]

**After attestation:** Delete this file.
