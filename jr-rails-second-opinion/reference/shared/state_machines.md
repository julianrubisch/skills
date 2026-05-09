# State Machines

## Summary

State machines describe possible states, transitions between them, and triggering events. They make implicit state logic explicit and centralized, preventing scattered conditional logic across the codebase.

## When to Use

- Multiple related boolean/timestamp attributes tracking state
- Complex conditional logic based on object state
- State-dependent behavior
- Audit trail requirements

## When NOT to Use

- Linear progressions (just use enum)
- Simple two-state flags
- Before patterns emerge (premature)

## Key Principles

- **Identify implicit state machines** — scattered booleans/timestamps often hide state
- **Prefer events over direct transitions** — `order.submit!` not `order.status = :submitted`
- **Extract when not central** — standalone state machine for secondary state
- **Guards and callbacks sparingly** — don't recreate callback problems

## Identifying Implicit State Machines

Signs of hidden state:

```ruby
# Multiple related attributes
class Post < ApplicationRecord
  # submitted_at, reviewed_at, published_at, archived_at
  # is_draft, is_approved, is_hidden
end

# Scattered conditionals
def can_publish?
  submitted_at.present? && reviewed_at.present? && !is_hidden
end

def publish!
  return unless can_publish?
  update!(published_at: Time.current)
  notify_subscribers if !was_published_before
end
```

## Implementation

### With Enum (Simple)

```ruby
class Post < ApplicationRecord
  enum :state, {
    draft: 0,
    submitted: 1,
    approved: 2,
    rejected: 3,
    published: 4,
    archived: 5
  }
end
```

### Pattern Matching Transitions

For cases where a gem isn't warranted:

```ruby
class Post < ApplicationRecord
  enum :state, %w[draft submitted approved rejected published archived].index_by(&:itself)

  def trigger(event)
    with_lock do
      case [state.to_sym, event.to_sym]
      in [:draft, :submit]
        update!(state: :submitted, submitted_at: Time.current)
      in [:submitted, :approve]
        update!(state: :approved, reviewed_at: Time.current)
      in [:submitted, :reject]
        update!(state: :rejected, reviewed_at: Time.current)
      in [:approved | :archived, :publish]
        update!(state: :published, published_at: Time.current, archived_at: nil)
      in [:published, :archive]
        update!(state: :archived, archived_at: Time.current)
      else
        false
      end
    end
  end
end

# Usage
post.trigger(:submit)
post.trigger(:approve)
post.trigger(:publish)
```

### With AASM

```ruby
class Post < ApplicationRecord
  include AASM

  aasm column: :state do
    state :draft, initial: true
    state :submitted
    state :approved
    state :rejected
    state :published
    state :archived

    event :submit do
      transitions from: :draft, to: :submitted
    end

    event :approve do
      transitions from: :submitted, to: :approved
    end

    event :reject do
      transitions from: :submitted, to: :rejected
    end

    event :revise do
      transitions from: :submitted, to: :draft
    end

    event :publish do
      transitions from: [:approved, :archived], to: :published,
                  after: -> { touch(:published_at) }
    end

    event :archive do
      transitions from: :published, to: :archived,
                  after: -> { touch(:archived_at) }
    end
  end
end

# Usage
post.submit!
post.may_approve?  #=> true
post.approve!
post.aasm.events   #=> [:publish]
```

### Transition Guards

```ruby
aasm column: :state do
  state :draft, initial: true

  event :publish do
    transitions from: :draft, to: :published, guard: :publishable?
  end

  event :fast_publish do
    transitions from: :draft, to: :published,
                guard: -> { user.karma >= MIN_TRUSTED_KARMA }
  end
end

def publishable?
  body.present? && title.present?
end
```

### Transition Callbacks

```ruby
aasm column: :state do
  event :approve, after: :record_review do
    transitions from: :submitted, to: :approved
  end
end

# Or use after_all_transitions for cross-cutting concerns
after_all_transitions :log_transition

private

def record_review
  self.reviewed_at = Time.current
  self.reviewed_by = Current.user
end

def log_transition
  Rails.logger.info "#{self.class}##{id}: #{aasm.from_state} → #{aasm.to_state}"
end
```

### After-Commit Callbacks

Use `after_commit` on events for side effects that should run after the transaction:

```ruby
aasm column: :state do
  event :approve, after_commit: :notify_author do
    transitions from: :submitted, to: :approved
  end
end

private

def notify_author
  ApprovalMailer.approved(self).deliver_later
end
```

### Standalone State Machine

When the state machine isn't central to the model, extract it into a separate
object. AASM can be included in any Ruby class:

```ruby
class Post::PublicationWorkflow
  include AASM

  attr_reader :post

  def initialize(post)
    @post = post
  end

  aasm do
    state :draft, initial: true
    state :submitted
    state :approved
    state :published

    event :submit do
      transitions from: :draft, to: :submitted
    end

    event :approve do
      transitions from: :submitted, to: :approved
    end

    event :publish, after_commit: :notify_subscribers do
      transitions from: :approved, to: :published
    end
  end

  private

  def aasm_read_state
    post.publication_state&.to_sym || :draft
  end

  def aasm_write_state(new_state)
    post.update!(publication_state: new_state.to_s)
  end

  def notify_subscribers
    PublicationMailer.published(post).deliver_later
  end
end

class Post < ApplicationRecord
  def publication_workflow
    @publication_workflow ||= PublicationWorkflow.new(self)
  end

  delegate :submit!, :approve!, :publish!, to: :publication_workflow
end
```

### Triggering Deliveries from State Machines

State machines are an excellent place to trigger notifications — they keep side
effects out of models while centralizing state-related behavior:

```ruby
class Cable < ApplicationRecord
  include AASM

  aasm column: :status do
    state :pending, initial: true
    state :creating
    state :created
    state :failed
    state :terminating

    event :create_cable do
      transitions from: [:pending, :failed], to: :creating
    end

    event :cable_created, after_commit: :notify_provisioned do
      transitions from: :creating, to: :created
    end

    event :error, after_commit: :notify_failure do
      transitions from: :creating, to: :failed
    end

    event :terminate do
      transitions from: :created, to: :terminating
    end
  end

  private

  def notify_provisioned
    CableDelivery.with(cable: self).provisioned.deliver_later
  end

  def notify_failure
    Admin::CableDelivery.with(cable: self).provision_failed.deliver_later
  end
end
```

**Benefits:**
- Keeps models free of scattered notification logic
- Notifications tied to state transitions, not arbitrary callbacks
- Easy to see all side effects for a given state change
- `after_commit` ensures notifications only fire after successful persistence

For multi-channel delivery (email + push + in-app), consider using the Noticed
gem — see `shared/notifications.md`. For complex cases with multiple subscribers
per transition, consider emitting events via `Rails.event.notify` (8.1+) from
`after_commit` callbacks — see `shared/instrumentation.md`.

## Events Over Direct Transitions

```ruby
# BAD: Direct state assignment
post.update!(state: :published)

# GOOD: Event-driven
post.publish!

# Why? Events:
# - Validate transition is allowed
# - Run guards and callbacks
# - Provide audit trail
# - Decouple layers
```

## Testing

```ruby
class PostStateMachineTest < ActiveSupport::TestCase
  test "draft can be submitted" do
    post = posts(:draft_post)

    assert post.may_submit?
    post.submit!
    assert_equal "submitted", post.state
  end

  test "submitted cannot be published directly" do
    post = posts(:submitted_post)

    assert_not post.may_publish?
    assert_raises(AASM::InvalidTransition) { post.publish! }
  end

  test "approved can be published" do
    post = posts(:approved_post)

    post.publish!
    assert_equal "published", post.state
    assert_not_nil post.published_at
  end

  test "guard prevents publishing without content" do
    post = posts(:approved_post)
    post.body = nil

    assert_not post.may_publish?
  end
end
```

## Anti-Patterns

### Implicit State Machines

```ruby
# BAD: State scattered across attributes
class Order < ApplicationRecord
  def status
    return :cancelled if cancelled_at?
    return :shipped if shipped_at?
    return :paid if paid_at?
    :pending
  end
end

# GOOD: Explicit state
class Order < ApplicationRecord
  include AASM

  aasm column: :status do
    state :pending, initial: true
    state :paid
    state :shipped
    state :cancelled

    event :pay do
      transitions from: :pending, to: :paid
    end
    # ...
  end
end
```

### Phantom Transitions

```ruby
# BAD: publish -> publish triggers side effects again
post.publish!  # Sends notifications
post.publish!  # Sends again!

# GOOD: AASM prevents this automatically — publish! raises
# AASM::InvalidTransition if already published
```

### Excessive Guards

```ruby
# BAD: Guard recreates business logic
event :approve do
  transitions from: :submitted, to: :approved,
              guard: -> { user.admin? && !flagged? && reviewed_by_two_people? }
end

# GOOD: Keep guards simple, logic in methods
event :approve do
  transitions from: :submitted, to: :approved, guard: :approvable?
end

def approvable?
  ApprovalPolicy.new(Current.user, self).allowed?
end
```

## Related Gems

| Gem | Purpose |
|-----|---------|
| [aasm](https://github.com/aasm/aasm) | State machines with ActiveRecord integration, guards, callbacks |
