# Standards the second opinion is judged against

The external CLI does not know this pack's conventions; without them it
returns generic Rails advice, or advice that contradicts the conventions
(service objects, factories, callbacks for side effects). This file is the
alignment layer: a short list per dimension, each item pointing at the
shared reference that explains it. Paste the **Brief block** at the end
into the CLI brief; use the full list yourself in Self-Review and
Reconcile.

Paths are relative to this skill's `reference/` symlink. When the CLI may
read files, give it absolute paths (`realpath reference/shared/callbacks.md`).

## Architecture

Judge against:

- Four layers, data flows one way, no reverse dependencies:
  `reference/shared/architecture.md` (Rules 1 to 3, Layer Mapping)
- No service objects; extraction goes to a domain model, form object, query
  object, concern, DCI context, or a method called explicitly:
  `reference/refactorings/010-refactor-service-object-into-poro.md`,
  `reference/refactorings/extraction-signals.md`
- Callbacks are scored: transformers, normalizers and utilities stay;
  operations (mail, jobs, cross-model writes) are extracted:
  `reference/shared/callbacks.md` (Callback Scoring System)
- Concerns for orthogonal behaviour, not for slicing one class into files:
  `reference/shared/concerns.md` (When NOT to Use, Concern Health Check)
- Authorization lives in policies and scopes, not inline in controllers:
  `reference/shared/authorization.md` (Layer Placement, Scoping-Based
  Authorization)
- Jobs are thin and idempotent; the model owns the operation:
  `reference/shared/jobs.md` (Idempotency)
- `Current.*` is read in controllers and passed in, never read inside
  model methods: `reference/smells.md` (Current in Models)

## Quality

Judge against:

- Tell, Don't Ask; CQS; SRP: `reference/shared/principles.md` (1, 3, and
  Command/Query Separation in `reference/smells.md`)
- Method and class smells with their thresholds: `reference/smells.md`
  (Long Method, Flag Arguments, God Class, Data Clumps)
- Law of Demeter, silent rescues, logic in views:
  `reference/anti-patterns.md` (Voyeuristic Models, Inaudible Failures,
  Logic in Views)
- Query logic belongs in scopes or query objects, not controllers:
  `reference/patterns.md` (Query Objects, Atomic vs Complex Scopes)
- A smell maps to a named refactoring; cite it: `reference/refactorings/`

## Performance

Judge against:

- N+1 on associations and ActiveStorage; eager load at the boundary that
  renders
- Every foreign key indexed; polymorphic pairs and uniqueness validations
  backed by a database index
- Filtering and sorting in SQL, not Ruby; `.size` over `.count` on loaded
  relations
- External calls have timeouts and error handling:
  `reference/anti-patterns.md` (Sluggish Services, Fire and Forget)
- Query objects for composition rather than ad-hoc chains:
  `reference/patterns.md`

## Testing

Judge against:

- Fixtures, not factories; mock only at system boundaries
- Integration tests for controllers; system tests only when a browser is
  needed
- Behaviour, not implementation; no `sleep`, use `travel`
- Concerns are tested through a host model, not in isolation:
  `reference/shared/concerns.md` (Testing Concerns)
- Jobs are tested for idempotency and retry: `reference/shared/jobs.md`

## Security

Judge against `reference/shared/security.md`, section by section: SQL
Injection, XSS Prevention, Strong Parameters, Authorization Scoping, SSRF
Protection, Command Injection, Path Traversal, Sensitive Data Exposure.
Severity follows the section's own ranking: injection and mass assignment
are H; XSS, IDOR, SSRF are H; CSP, session config, open redirects are M.

## Brief block

Paste verbatim into the brief under `STANDARDS`. Keep it this short; the
CLI needs the rules, not the rationale.

```
STANDARDS (this project's conventions; findings that contradict them are
discarded, findings that cite them are preferred):
- Layers: presentation -> application -> domain -> infrastructure, one way.
- No service objects. Extraction targets: domain model, form object, query
  object, concern, DCI context, explicit method call.
- Callbacks: keep transformers/normalizers; extract operations (mail, jobs,
  writes to other models) into explicitly called methods.
- Concerns share orthogonal behaviour; they do not slice one class into files.
- Authorization in policies and scopes, never inline find(params[:id]).
- Current.* is read in controllers and passed in, never inside models.
- Jobs are thin and idempotent; the model owns the operation.
- Query logic in scopes or query objects, not in controllers.
- Tests: fixtures not factories; mock only at system boundaries;
  integration tests for controllers; system tests only when a browser is
  needed; no sleep.
- Security: judge injection, mass assignment, XSS, IDOR, SSRF as H; CSP,
  session config, open redirects as M.
- Name the smell and the refactoring when you flag one.
```
