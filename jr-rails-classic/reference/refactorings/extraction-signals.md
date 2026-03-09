# Extraction Signals

How to identify code that should be extracted to a different layer or abstraction.
This is the triage guide — individual refactoring files say *how*, this says *when*.

## Callback Scoring System

> See `shared/callbacks.md` for the full callback reference with detailed
> examples, anti-patterns, and extraction walkthrough.

Rate model callbacks to identify extraction candidates:

| Score | Type | Description | Action |
|-------|------|-------------|--------|
| 5/5 | **Transformer** | Computes/defaults required values | Keep in model |
| 4/5 | **Normalizer** | Sanitizes input data | Keep (prefer `.normalizes` API) |
| 4/5 | **Utility** | Counter caches, cache busting | Keep in model |
| 2/5 | **Observer** | Side effects after commit | Review case-by-case |
| 1/5 | **Operation** | Business process steps | Extract immediately |

### Transformer Callbacks (Keep)

Compute or default required attribute values:

```ruby
class Post < ApplicationRecord
  before_validation :compute_shortname, on: :create
  before_save :set_word_count, if: :content_changed?

  private

  def compute_shortname
    self.short_name ||= title.parameterize
  end

  def set_word_count
    self.word_count = content.split(/\s+/).size
  end
end
```

### Normalizer Callbacks (Keep)

Sanitize user input. Prefer Rails 7.1+ `.normalizes` API:

```ruby
class Post < ApplicationRecord
  normalizes :title, with: -> { _1.strip }
  normalizes :content, with: -> { _1.squish }
end
```

### Utility Callbacks (Keep)

Framework-level utilities like counter caches:

```ruby
class Comment < ApplicationRecord
  belongs_to :post, touch: true, counter_cache: true
end
```

### Operation Callbacks (Extract)

Signs of misplacement:
- Conditions (`unless: :admin?`)
- Collaboration with non-model objects (mailers, API clients)
- Remote peer communication

```ruby
# BAD - Extract these
class User < ApplicationRecord
  after_create :generate_initial_project, unless: :admin?
  after_commit :send_welcome_email, on: :create
  after_commit :sync_with_crm
  after_commit :track_signup_analytics
end
```

**Extraction options** (see `009-replace-callback-with-method.md`, `004-introduce-form-object.md`):
1. Move to controller (simplest)
2. Move to form object (when the callback is part of a user interaction flow)
3. Use an event-driven approach (when multiple subscribers need to react)

### Event-driven extraction

> See `shared/instrumentation.md` for the full event pipeline reference —
> `Rails.event` API, subscriber patterns, tags/context, and testing.

#### Rails 8.1+: `Rails.event` (ActiveSupport::EventReporter)

Built-in structured event bus. **In-process only** — subscribers run in the
same process, so side effects should dispatch to background jobs for anything
heavy or external.

```ruby
# Model emits a structured event
class User < ApplicationRecord
  after_commit on: :create do
    Rails.event.notify("user.created", { id: id, email: email })
  end
end

# Subscriber handles side effects — decoupled from the model
class UserEventSubscriber
  def emit(event)
    case event[:name]
    when "user.created"
      UserMailer.welcome(event[:payload][:id]).deliver_later
      CrmSyncJob.perform_later(event[:payload][:id])
    end
  end
end

# config/initializers/event_subscribers.rb
Rails.event.subscribe(UserEventSubscriber.new) { |e| e[:name].start_with?("user.") }
```

**Ops note:** `Rails.event` is in-process — no Redis or external backend
required, but also no cross-process delivery. For distributed event handling,
subscribers should enqueue jobs (Solid Queue / Sidekiq) that do the actual work.

#### Rails ≤ 8.0: ActiveSupport::Notifications

Same pattern using the older notifications API:

```ruby
class User < ApplicationRecord
  after_commit on: :create do
    ActiveSupport::Notifications.instrument("user.created", user: self)
  end
end

# config/initializers/event_subscribers.rb
ActiveSupport::Notifications.subscribe("user.created") do |event|
  UserMailer.welcome(event.payload[:user]).deliver_later
  CrmSyncJob.perform_later(event.payload[:user].id)
end
```

## God Object Identification

### Churn × Complexity Metric

**Churn** = how often a file changes (indicates ongoing modifications)
**Complexity** = code complexity score (use Flog)

Files high in both are prime refactoring candidates.

```bash
# Calculate churn
git log --format=oneline -- app/models/user.rb | wc -l

# Calculate complexity
flog -s app/models/user.rb

# Find intersection of top 10 by each
```

### Automated Tool

Use [attractor](https://github.com/julianrubisch/attractor):

```bash
attractor report -p app/models
```

### Common God Object Names

Watch for these accumulating responsibilities:
- `User` / `Account`
- `Order` / `Transaction`
- `Project` / `Workspace`
- `Post` / `Article`

### Decomposition Strategies

1. **Extract concerns** for shared behaviors
2. **Extract delegate objects** for complex operations. Use https://github.com/kaspth/active_record-associated_object
3. **Extract value objects** for groups of related attributes
4. **Create new models** for distinct concepts

```ruby
# Before: God User model
class User < ApplicationRecord
  # Authentication (20 methods)
  # Profile (15 methods)
  # Notifications (10 methods)
  # Analytics (10 methods)
end

# After: Decomposed
class User < ApplicationRecord
  include User::Authentication
  has_one :profile
  has_one :notification_preferences
end

class User::Authentication
  # Authentication behavior
end

class Profile < ApplicationRecord
  belongs_to :user
end

class NotificationPreferences < ApplicationRecord
  belongs_to :user
end
```

## Concern Health Check

### Good Concerns (Behavioral)

Can be tested in isolation, shared across models:

```ruby
module Publishable
  extend ActiveSupport::Concern

  included do
    scope :published, -> { where.not(published_at: nil) }
    scope :draft, -> { where(published_at: nil) }
  end

  def published? = published_at.present?
  def publish! = update!(published_at: Time.current)
end
```

**Test:** Can you write tests for this concern without instantiating the host model?

### Bad Concerns (Code-Slicing)

Groups code by Rails artifact type, not behavior:

```ruby
# BAD - Just groups contact-related code
module Contactable
  extend ActiveSupport::Concern

  included do
    validates :email, presence: true
    validates :phone, format: { with: PHONE_REGEX }
    before_save :normalize_phone
  end

  def full_contact_info
    "#{email} / #{phone}"
  end
end
```

**Test:** If removing this concern breaks unrelated tests, it's code-slicing.

### Overgrown Concerns

Signs a concern should be extracted:
- 50+ lines
- Multiple responsibilities
- Complex internal state

Extract to:
- **Delegate object** for operations
- **Value object** for attribute groups
- **Separate model** for distinct entity


## Anemic Model Warning Signs

Your models might be anemic if:
- Services contain calculations that use only model data
- Models are pure data containers (associations + validations only)
- You have `CalculateXService` for model attributes
- Domain rules live in services, not models

```ruby
# BAD - Anemic
class Order < ApplicationRecord
  # Just associations and validations
end

class CalculateOrderTotalService
  def call(order)
    order.items.sum { |i| i.price * i.quantity }
  end
end

# GOOD - Rich model
class Order < ApplicationRecord
  def total
    items.sum(&:subtotal)
  end
end
```

## Controller Fat Signals

### Extract When You See

- Business calculations (pricing, discounts)
- Multiple model updates
- Complex conditionals based on business rules
- External API calls
- More than 10-15 lines per action

### Keep in Controller

- Parameter parsing
- Authentication/authorization
- Response formatting
- Simple model operations

## Signal → Refactoring Map

When you spot one of these signals, reach for the corresponding refactoring:

| What you see | Refactoring |
|-------------|-------------|
| `case`/`if-elsif` branching on object type | `002` Replace Conditional with Polymorphism |
| Repeated `nil` checks / `try` / `&.` for absent values | `003` Replace Conditional with Null Object |
| Callback with side effects, conditional callbacks, multi-model orchestration in callbacks | `004` Introduce Form Object |
| STI subclasses varying on one behavioral dimension, can't switch type without recreate | `005` Replace Subclasses with Strategies |
| Mixin with business logic, name clashes, hard to test without host model | `006` Replace Mixin with Composition |
| Complex regex/multi-step validation, same validation on 2+ models | `007` Extract Validator |
| Method with 3+ related args that always appear together (data clump) | `008` Introduce Parameter Object |
| Callback sending email, creating records in other models, calling external APIs | `009` Replace Callback with Method |
| Class named `*Service`/`*Manager`/`*Handler` with a `.call` method | `010` Rename Service Object to Domain Model |
| `.where`/`.order` chains repeated in controllers or across contexts | `001` Extract Scope from Controller |
| Transient attributes, UI-flow-specific validation, virtual fields on a model | `004` Introduce Form Object |
| Same 2-3 primitive values always traveling together | `008` Introduce Parameter Object + Value Object (`composed_of`) |
| Helper with heavy `tag.*` chains, complex data attributes, or nested yielding blocks | Extract to Phlex component (`shared/components.md § Extraction Signals`) |
| Presenter building HTML via `content_tag`/`tag.*` | Extract to Phlex component (`shared/components.md § From Presenters`) |

## Quick Reference: Thresholds

| Signal | Threshold | Action |
|--------|-----------|--------|
| Callback score | ≤ 2/5 | Extract to form object / event |
| Model complexity | Flog > 100 | Decompose (concerns, value objects, delegate objects) |
| Model churn | > 30 changes/year | Review for extraction |
| Concern size | > 50 lines | Extract to delegate object or separate model |
| Controller action | > 15 lines | Extract to domain model / form object |
| Service with domain logic | Any calculations | Move to model (anemic model fix) |

## Tools

| Tool | Purpose | Command |
|------|---------|---------|
| [flog](https://github.com/seattlerb/flog) | Complexity scoring | `flog -s app/models/` |
| [attractor](https://github.com/julianrubisch/attractor) | Churn × complexity | `attractor report` |
| [callback_hell](https://github.com/evilmartians/callback_hell) | Callback audit | `bin/rails callback_hell:callbacks[User]` |
