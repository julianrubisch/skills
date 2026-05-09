# Ruby Science Principles

Design principles that review agents cite when flagging smells. Ordered by
priority — higher-priority principles address more common Rails anti-patterns.

## 1. Tell, Don't Ask

Ask an object to do something, don't extract its data and do it yourself.
Code that queries an object's state to make decisions on its behalf creates
Feature Envy and violates CQS (asking then telling in the same flow).

In Rails: move conditional logic into the model that owns the data. If a
controller checks `@order.paid? && @order.shipped?` before acting, the order
should expose `@order.fulfillable?` or `@order.fulfill!` instead.

## 2. Composition over Inheritance

Favor composing behavior from collaborators over inheriting from base classes.
Inheritance creates tight coupling and makes it hard to vary one dimension
without affecting others. In Rails, this shows up as deep STI hierarchies,
god concerns, and base classes that accumulate unrelated responsibilities.

Prefer: mixins for "acts-as" behavior, delegation for "uses-a" relationships,
strategy objects for interchangeable algorithms. Inherit only for genuine
"is-a" taxonomies.

## 3. Single Responsibility Principle (SRP)

A class should have one reason to change. When a model handles validation,
notification, reporting, and third-party sync, any change to one concern risks
breaking the others. The signal: a class file that changes in every PR, for
unrelated reasons (Divergent Change).

In Rails: extract concerns for orthogonal behaviors, form objects for UI-specific
validation, query objects for complex reads, and event-driven side effects
via `Rails.event` instead of inline callbacks.

## 4. Dependency Inversion Principle (DIP)

Depend on abstractions, not concretions. High-level business logic should not
depend directly on low-level infrastructure (mailers, external APIs, storage).
The signal: a model that `require`s an API client or calls `Net::HTTP` directly
(Inappropriate Intimacy with infrastructure).

In Rails: inject dependencies via constructor arguments or use `Rails.event` /
`ActiveSupport::Notifications` to decouple the trigger from the handler. Jobs
and notifiers are natural abstraction boundaries.

## 5. Open/Closed Principle (OCP)

Classes should be open for extension, closed for modification. When adding a
new case requires editing existing `case`/`if` chains, OCP is violated. The
signal: Long Case Statements that grow with every new variant.

In Rails: use polymorphism (`delegated_type`), strategy objects, or STI with
overridden methods. Each new variant is a new class, not a new branch.

## 6. Liskov Substitution Principle (LSP)

Subtypes must be substitutable for their base types without altering correctness.
If code needs to check `is_a?` or `respond_to?` before calling a method, the
subtype contract is broken. The signal: conditional logic that branches on class
type instead of trusting polymorphism.

In Rails: STI subclasses must honor the parent's interface — if `Animal#speak`
returns a string, `Dog#speak` must too (not raise or return nil). When using
`delegated_type`, each delegated class must implement the full interface the
delegator expects. Concerns that override methods must preserve the return type
and side-effect contract of the original.

## 7. Interface Segregation Principle (ISP)

Clients should not be forced to depend on methods they don't use. Fat interfaces
that bundle unrelated capabilities create coupling to things the caller doesn't
care about. The signal: classes that include a concern but only use 1 of its 10
methods, or collaborators that receive a full ActiveRecord model when they only
need two attributes.

In Rails: split god concerns into focused ones — `Searchable`, `Publishable`,
`Auditable` instead of one `Featureable` concern. Pass value objects or structs
to collaborators instead of entire models. When designing POROs, keep the public
interface to what callers actually need.

## 8. Law of Demeter (LoD)

Only talk to your immediate collaborators — don't chain through objects to
reach their internals. `@order.customer.address.city` couples you to the
entire object graph. The signal: long method chains, especially across
association boundaries (Inappropriate Intimacy).

In Rails: use `delegate` for commonly accessed nested attributes, or expose
a domain method that encapsulates the traversal.

## 9. DRY (Don't Repeat Yourself)

Every piece of knowledge should have a single, authoritative representation.
Duplication isn't just identical code — it's identical *intent* expressed in
multiple places. The signal: changing one behavior requires updating 3+ files
(Shotgun Surgery).

In Rails: extract shared logic into concerns, base classes, or shared modules.
But avoid premature DRY — three similar lines are fine if they represent
different domain concepts that may diverge later.

---

## Principle → Smell Mapping

Review agents use this table to cite the violated principle when flagging a smell.

| Principle | Related Smells |
|-----------|---------------|
| Tell, Don't Ask | Feature Envy, CQS Violation |
| Composition over Inheritance | Large Class, Callback, Long Case Statement |
| SRP | Divergent Change, Large Class, Callback |
| DIP | Feature Envy, Inappropriate Intimacy |
| OCP | Case Statement, Long Case Statement |
| LSP | Case Statement, Refused Bequest, Type-Checking Dispatch |
| ISP | Large Class, God Class, Feature Envy |
| Law of Demeter | Inappropriate Intimacy, Feature Envy |
| DRY | Shotgun Surgery, Divergent Change |
