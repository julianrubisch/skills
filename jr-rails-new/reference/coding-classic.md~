# Coding Style: Classic / Adapted 37signals Rails

## Philosophy

> "The best code is the code you don't write. The second best is the code that's obviously correct."

**Vanilla Rails is plenty:**
- Rich domain models over service objects
- CRUD controllers over custom actions
- Concerns for horizontal code sharing
- State as records instead of boolean columns
- Database-backed everything (no Redis)

**Development approach:**
- Ship, Validate, Refine — prototype-quality code to production to learn
- Fix root causes, not symptoms
- Write-time operations over read-time computations
- Let it crash: use bang methods, let Rails handle RecordInvalid with 422s

---

## Naming

**Verbs** for state changes: `card.close`, `card.gild`, `board.publish` (not `set_style`)

**Predicates** derived from record presence: `card.closed?`, `card.golden?`

**Concerns** named as adjectives: `Closeable`, `Publishable`, `Watchable`

**Controllers** as nouns matching resources: `Cards::ClosuresController`

**Scopes:**
```ruby
scope :chronologically,         -> { order(created_at: :asc) }
scope :reverse_chronologically, -> { order(created_at: :desc) }
scope :alphabetically,          -> { order(title: :asc) }
scope :latest,                  -> { reverse_chronologically.limit(10) }
scope :preloaded,               -> { includes(:creator, :assignees, :tags) }
scope :active,                  -> { ... }   # business terms, not SQL-ish
scope :indexed_by,              ->(col) { order(col => :asc) }
```

---

## REST Mapping

No custom actions. Create new resources instead:

| Action | Old | New |
|--------|-----|-----|
| close a card | `POST /cards/:id/close` | `POST /cards/:id/closure` |
| archive a card | `POST /cards/:id/archive` | `POST /cards/:id/archival` |
| watch a board | — | `POST /boards/:id/watching` |
| mark as golden | — | `POST /cards/:id/goldness` |

```ruby
resources :boards do
  resources :cards do
    resource :closure     # singular: one per card
    resource :goldness
    resource :not_now
    resources :assignments
    resources :comments
  end
end

# Shallow nesting to avoid deep URLs
resources :boards do
  resources :cards, shallow: true   # /boards/:id/cards, but /cards/:id
end

# Resolve for correct URL generation
resolve("Comment") { |comment| [comment.card, anchor: dom_id(comment)] }
```

---

## Models

### State as Records, Not Booleans

Instead of `closed: boolean`, create a record. Gives you timestamps, authorship,
and clean joins for free.

```ruby
class Card::Closure < ApplicationRecord
  belongs_to :card
  belongs_to :creator, class_name: "User"
end

module Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
  end

  def closed? = closure.present?

  def close(creator: Current.user)
    create_closure!(creator: creator)
  end

  def reopen
    closure&.destroy
  end
end

# Querying
Card.joins(:closure)          # closed
Card.where.missing(:closure)  # open
```

### Concerns for Horizontal Behavior

Each concern is self-contained (associations, scopes, methods). 50–150 lines.
Named for capabilities, not organization.

```ruby
class Card < ApplicationRecord
  include Assignable
  include Closeable
  include Golden
  include Watchable
  include Searchable
  # ...
end
```

### POROs Namespaced Under Models

Business logic that doesn't fit a concern goes in a PORO under the model namespace.
Not service objects — domain objects with clear responsibility.

```ruby
# app/models/event/description.rb
class Event::Description
  def initialize(event) = @event = event
  def to_s = # ...
end

# app/models/card/eventable/system_commenter.rb
class Card::Eventable::SystemCommenter
  def initialize(card) = @card = card
  def comment(message) = # ...
end
```

### Callbacks — Used Sparingly

Only for derived data and async side effects. Not for business logic or synchronous external calls.

```ruby
class Card < ApplicationRecord
  after_create_commit :notify_watchers_later
  before_save :update_search_index, if: :title_changed?
end
```

### Validations

Minimal on the model (data integrity). Contextual on form objects (UI flows).
Prefer database constraints over model validations for hard guarantees.

```ruby
# Model — minimal
class User < ApplicationRecord
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  normalizes :email, with: ->(e) { e.strip.downcase }   # Rails 7.1+
end

# Form object — contextual
class Signup
  include ActiveModel::Model
  attr_accessor :email, :name, :terms_accepted
  validates :email, :name, presence: true
  validates :terms_accepted, acceptance: true

  def save
    return false unless valid?
    User.create!(email:, name:)
  end
end

# Migration — hard constraints
add_index :users, :email, unique: true
add_foreign_key :cards, :boards
```

### Rails 7.1+ Patterns

```ruby
# Delegated types (replace polymorphic associations)
class Message < ApplicationRecord
  delegated_type :messageable, types: %w[Comment Reply Announcement]
end

# store_accessor for structured JSON/JSONB — prefer over store()
class User < ApplicationRecord
  store_accessor :settings, :theme, :notifications_enabled
end
```

### Database

```ruby
# UUIDs as primary keys (UUIDv7 — time-sortable)
create_table :cards, id: :uuid do |t|
  t.references :board, type: :uuid, foreign_key: true
end

# Counter caches
class Comment < ApplicationRecord
  belongs_to :card, counter_cache: true
end

# Default values with Current
class Card < ApplicationRecord
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end
```

---

## Controllers

### Concerns for Shared Behavior

```ruby
module CardScoped
  extend ActiveSupport::Concern

  included do
    before_action :set_card
  end

  private
    def set_card
      @card  = Card.find(params[:card_id])
      @board = @card.board
    end

    def render_card_replacement
      render turbo_stream: turbo_stream.replace(@card)
    end
end

# Common controller concerns:
# BoardScoped, CardScoped, CurrentRequest, CurrentTimezone,
# SetPlatform, FilterScoped, TurboFlash, ViewTransitions,
# BlockSearchEngineIndexing
```

### Turbo Stream Responses

```ruby
class Cards::ClosuresController < ApplicationController
  include CardScoped

  def create  = @card.close   && render_card_replacement
  def destroy = @card.reopen  && render_card_replacement
end

# For complex updates:
render turbo_stream: turbo_stream.morph(@card)
```

---

## Current Attributes

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :account, :request_id

  delegate :user, to: :session, allow_nil: true

  def account=(account)
    super
    Time.zone = account&.time_zone || "UTC"
  end
end

# Set in ApplicationController
before_action :set_current_request

def set_current_request
  Current.session    = authenticated_session
  Current.account    = Account.find(params[:account_id])
  Current.request_id = request.request_id
end
```

---

## Background Jobs

Jobs are thin wrappers over model methods. Logic lives on the model.

```ruby
# _later / _now naming convention on the concern
module Watchable
  def notify_watchers_later = NotifyWatchersJob.perform_later(self)
  def notify_watchers_now   = NotifyWatchersJob.perform_now(self)

  def notify_watchers
    watchers.each { |w| WatcherMailer.notification(w, self).deliver_later }
  end
end

# Job is a shallow dispatcher
class NotifyWatchersJob < ApplicationJob
  def perform(card) = card.notify_watchers
end

# Enqueue after transaction commit (Rails 7.2+)
# config/application.rb
config.active_job.enqueue_after_transaction_commit = true
```

---

## Email

```ruby
# Timezone-aware delivery
class NotificationMailer < ApplicationMailer
  def daily_digest(user)
    Time.use_zone(user.timezone) do
      @digest = user.digest_for_today
      mail(to: user.email)
    end
  end
end

# Batch delivery
ActiveJob.perform_all_later(users.map { |u| NotificationMailer.digest(u).deliver_later })

# One-click unsubscribe (RFC 8058)
class ApplicationMailer < ActionMailer::Base
  after_action :set_unsubscribe_headers

  private
    def set_unsubscribe_headers
      headers["List-Unsubscribe-Post"] = "List-Unsubscribe=One-Click"
      headers["List-Unsubscribe"]      = "<#{unsubscribe_url}>"
    end
end
```

---

## Active Storage

```ruby
# Variant preprocessing
has_one_attached :avatar do |a|
  a.variant :thumb,  resize_to_limit: [100, 100],  preprocessed: true
  a.variant :medium, resize_to_limit: [300, 300],  preprocessed: true
end

# Extend expiry for slow connections
config.active_storage.service_urls_expire_in = 48.hours

# Avatar show action — redirect to blob for long-term caching
def show
  expires_in 1.year, public: true
  redirect_to @user.avatar.variant(:thumb).processed.url, allow_other_host: true
end
```

---

## Testing

Preferences only — full testing guide in `shared/testing.md`.

- **Minitest**, not RSpec
- **Fixtures**, not factory_bot
- **Integration tests** for controllers, not controller tests
- Test observable behavior, not implementation
- Don't mock what you can test for real
- VCR for external APIs

---

## Preferred Stack

| Concern | Gem |
|---------|-----|
| Frontend | turbo-rails, stimulus-rails, importmap-rails |
| Assets | propshaft |
| Testing | Minitest |
| Jobs | Solid Queue (no Redis) |
| Cache/Cable | Solid Cache, Solid Cable |
| Authorization | Pundit |
| Rich text | Lexxy |
| Mailer utils | Mittens |
| Deployment | Kamal + Thruster |
| Job monitoring | Mission Control Jobs |
| GC tuning | Autotuner |

---

## Gem Selection Heuristics

1. **Can vanilla Rails do this?** — ActiveRecord, ActionMailer, ActiveJob cover most needs
2. **Is it the app's core concern?** — if yes, own the code; if fringe, use a gem
3. **Does it add infrastructure?** — Redis? Database-backed alternatives exist
4. **Is the complexity worth it?** — 150 lines of custom code vs. a 10k-line gem
5. **Is it from someone you trust?** — 37signals gems are battle-tested at scale

> "Build solutions before reaching for gems."

---

## Frontend

<!-- Rails-side integration only. hwc-* skills cover Stimulus/Turbo mechanics. -->
<!-- Pairs with: ERB + Hotwire + Stimulus (conventional stack) -->

<!-- Add: respond_to turbo_stream conventions, Turbo Frame naming, stream naming, etc. -->
