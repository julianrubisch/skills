# Code Smells

Detection signals for review agents. A smell doesn't mean the code is wrong —
it means "look closer here." Use these to find candidates for deeper diagnosis.

## Method-level Smells

### Long Method
A method that does more than one thing. Hard to name precisely.

**Signal:** More than ~10 lines, or you need "and" to describe what it does.

### Long Parameter List
Method signature with 3+ positional arguments.

**Signal:** `def create(user, plan, coupon, trial_days)` — callers must know argument order.
Keyword arguments help but don't fix the underlying problem.

**Fix:** [Introduce Parameter Object](refactorings/008-introduce-parameter-object.md)

_[Ruby Science →](https://thoughtbot.com/ruby-science/long-parameter-list.html)_

### Flag Arguments
Boolean passed to change method behavior — the method is doing two things.

```ruby
# Smell: what does `true` mean at the call site?
send_notification(user, true)

# Better: two methods, or a keyword that reads clearly
send_notification(user, immediately: true)
```

## Class-level Smells

### God Class
**Pattern**: Class that knows too much about the system.

**Detection**:
- References to most other models
- Difficult to answer questions without this class
- Very high number of methods and lines
- Common in `User` model or central domain object

**Severity**: Critical

**Solutions**:
- Extract Class aggressively
- Use composition
- Introduce domain-specific objects

See `refactorings/extraction-signals.md § God Object Identification` for
churn × complexity metrics and decomposition strategies.

### Large Class
Class with too many responsibilities. Hard to name without "and" or "manager".

**Signal:** 200+ lines, 10+ public methods, multiple unrelated instance variables.

**Fix:** [Introduce Form Object](refactorings/004-introduce-form-object.md), [Replace Mixin with Composition](refactorings/006-replace-mixin-with-composition.md)

_[Ruby Science →](https://thoughtbot.com/ruby-science/large-class.html)_

### Data Clumps
Same group of data always appearing together — they probably want to be an object.

```ruby
# Smell: address fields scattered everywhere
def ship(street, city, zip, country)
def validate_address(street, city, zip, country)

# Signal: extract Address value object
```

**Fix:** Extract a Value Object (see `patterns.md § Value Objects`). For
denormalized columns on a single table, use `composed_of` to map columns
to the value object without a migration.

## Interaction Smells

### Command/Query Separation Violation
A method that both returns information *and* mutates state. Asking a question
should not change the answer.

```ruby
# Smell: name implies a query, but it also updates a property
def memory_lookup(user_message)
  observations_processed = []
  # ... queries memory ...
  update!(memory_property: result)  # hidden mutation!
  result
end

# Better: separate the query from the command
def memory_for(user_message)   # pure query
def store_memory!(observations) # explicit command
```

**Signal:** A method named as a query (`find_`, `get_`, `lookup_`, `for_`) that
also calls `update!`, `save`, `create`, or `increment!`.

### Model Creating/Destroying Other Models
A model method that creates, updates, or destroys records in unrelated models.
Increases coupling and violates SRP — managing its own associations is a
fringe exception, not a license to orchestrate broadly.

```ruby
# Smell: User model orchestrating Workspace and Notification creation
def onboard!
  Workspace.create!(owner: self)
  AdminNotification.create!(event: :new_user, subject: self)
end

# Better: double dispatch to a job, or DCI context
def onboard!
  OnboardingJob.perform_later(self)
end
```

**Signal:** Model methods calling `OtherModel.create!`, `other_record.update!`,
or `OtherModel.destroy_by(...)` on records outside their own association graph.

### Current in Models
Accessing `Current.user`, `Current.account` etc. *inside a method body* is a
hidden dependency on the request context. Silent failures in jobs and background
threads where Current is not set.

```ruby
# Smell — buried, non-overridable dependency
def process!
  log_action(Current.user)   # caller can't inject a different user
end

# Fine — caller can always pass an explicit value
def close(creator: Current.user)
  create_closure!(creator:)
end

# Fine — belongs_to convenience default
belongs_to :creator, class_name: "User", default: -> { Current.user }
```

**Signal:** `Current.*` used *inside* a method body with no way for the caller
to override it. Default arguments and `belongs_to` defaults are acceptable.
See `shared/current_attributes.md` for acceptable uses, multi-tenancy, and testing patterns.

### Feature Envy
A method that's more interested in another object's data than its own.

```ruby
# Smell: Order method obsessed with customer data
def apply_discount
  if customer.membership.tier == :gold && customer.membership.years > 2
    self.discount = 0.2
  end
end
# Signal: this logic might belong on Customer or Membership
```

**Fix:** Move the method to the object it envies, or [Extract Validator](refactorings/007-extract-validator.md) / [Introduce Form Object](refactorings/004-introduce-form-object.md) if it belongs in a dedicated object.

See also: `anti-patterns.md § Voyeuristic Models` for Law of Demeter violations
(3+ level association chains in views/controllers).

_[Ruby Science →](https://thoughtbot.com/ruby-science/feature-envy.html)_

### Inappropriate Intimacy
Two classes know too much about each other's internals.

**Signal:** Class A reaches into Class B's associations or private state directly.
Often indicates a missing abstraction between them.

### Type-Checking Dispatch
Branching on an object's class outside of `app/models/` — the code is doing
the object's job instead of asking the object to do it.

```ruby
# Smell: controller branches on type to build a URL
case item
when Project then project_path(item)
when Task    then project_task_path(item.project, item)
when Comment then polymorphic_path(item.commentable, anchor: dom_id(item))
end

# Better: presenter on each type owns its URL
item.presenter.path
```

**Signal:**
- `is_a?(...)`, `kind_of?(...)`, `.class == ...` in any file outside `app/models/`
- `case obj; when Foo` dispatching to different behavior per type
- More than one `if X.is_a?` branch on the same variable in the same method

**Where to push the behavior:**

| Where the behavior belongs | Push it to |
|----------------------------|------------|
| Pure domain logic (validation, state transitions, computed data) | Concern on the model |
| View/URL/formatting concern (what `url_helpers`, `helpers` touches) | Presenter (see `patterns.md § Presenters`) |
| Behavior varies on a persisted attribute, not the class itself | `delegated_type` or state machine |

**Fix:** [Replace Conditional with Polymorphism](refactorings/002-replace-conditional-with-polymorphism.md) —
see § "Where to Put the Extracted Behavior" for the decision table.

**Principle:** LSP — if code checks `is_a?` before calling a method, the
subtype contract is broken. See `shared/principles.md § Liskov Substitution`.

### Case Statement
A `case` with many `when` branches handling unrelated behavior. Attractive
for assembling all conditions in one place, but becomes a maintenance burden.

**Signal:** `case` with 5+ branches, or branches that call out to different
objects/concerns. Often a candidate for pattern matching, a lookup table,
or composition (e.g. strategies).

**Fix:** [Replace Conditional with Polymorphism](refactorings/002-replace-conditional-with-polymorphism.md), [Replace Conditional with Null Object](refactorings/003-replace-conditional-with-null-object.md)

_[Ruby Science →](https://thoughtbot.com/ruby-science/case-statement.html)_

## Change Smells

### Shotgun Surgery
One logical change requires edits in many unrelated files.

**Signal:** Adding a new payment provider touches 6 files. The concept isn't encapsulated.

**Fix:** [Replace Conditional with Polymorphism](refactorings/002-replace-conditional-with-polymorphism.md), [Introduce Parameter Object](refactorings/008-introduce-parameter-object.md)

_[Ruby Science →](https://thoughtbot.com/ruby-science/shotgun-surgery.html)_

### Divergent Change
One class changes for many different reasons.

**Signal:** "I edit this file whenever we change billing logic AND whenever we change
notification logic." The class has multiple axes of change — split it.

**Fix:** [Replace Conditional with Polymorphism](refactorings/002-replace-conditional-with-polymorphism.md), [Replace Subclasses with Strategies](refactorings/005-replace-subclasses-with-strategies.md)

_[Ruby Science →](https://thoughtbot.com/ruby-science/divergent-change.html)_

### Callback
A Rails lifecycle callback (`after_create`, `after_save`, etc.) that performs
work unrelated to the record's own persistence — sending email, creating records
in other models, calling external services. The side effect is invisible at the
call site and fires even when triggered by an unrelated save elsewhere.

```ruby
# Smell: delivery coupled to persistence
class Invitation < ApplicationRecord
  after_create :deliver   # fires on every create, even in tests/seeds

  private
  def deliver = InvitationMailer.invite(self).deliver_later
end
```

**Signal:** Callbacks containing business logic (payments, notifications, cross-model
creation). Methods like `save_without_sending_email` that exist to circumvent a callback.
Conditionally-invoked callbacks that only apply in some contexts.

**Fix:** [Replace Callback with Method](refactorings/009-replace-callback-with-method.md), [Introduce Form Object](refactorings/004-introduce-form-object.md).
For multiple subscribers: event bus (`Rails.event` 8.1+ / `ActiveSupport::Notifications`).
See `refactorings/extraction-signals.md § Event-driven extraction` and
`shared/callbacks.md` for the scoring system and extraction walkthrough.

**Exception:** Own-state callbacks are fine — `before_save :normalize_email`,
`before_save :update_search_index, if: :title_changed?`. Async job dispatch is
also acceptable — `after_create_commit :notify_later`.

_[Ruby Science →](https://thoughtbot.com/ruby-science/callback.html)_
