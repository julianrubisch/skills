---
name: jr-rails-classic
description: >-
  Write Rails code in 37signals/classic style: rich models, CRUD controllers,
  concerns, state-as-records, Minitest. Use when writing or modifying Ruby on
  Rails application code.
---

# Rails Classic Coding Style

Write Rails application code following 37signals conventions. Rich domain
models, CRUD controllers, database-backed everything, Minitest with fixtures.

## Core Workflow

1. **Use generators first** — `rails g model`, `rails g controller`,
   `rails g migration`. Generators produce correct file structure, test stubs,
   and route entries in one shot.
2. **Implementation order** — models → controllers → views → tests.
3. **Ship, Validate, Refine** — prototype to production, learn from real usage.
4. **Let it crash** — use bang methods, let Rails handle RecordInvalid with 422s.

## Guardrails

- No service objects — use domain models namespaced under `app/models/`
- No custom controller actions — create new resources instead (REST mapping)
- No RSpec, no factory_bot — Minitest with fixtures only
- No Redis — Solid Queue, Solid Cache, Solid Cable
- No `strftime` in views — custom DATE_FORMATS in initializers
- Callbacks only for derived data and async dispatch — never business logic
- Database constraints over model validations for hard guarantees
- Pass IDs to jobs, not objects

## Conventions Quick Reference

### Naming
- **Verbs** for state changes: `card.close`, `board.publish`
- **Predicates** from record presence: `card.closed?`, `card.golden?`
- **Concerns** as adjectives: `Closeable`, `Publishable`, `Watchable`
- **Controllers** as nouns: `Cards::ClosuresController`
- **Scopes** as business terms: `scope :active`, `scope :chronologically`

### REST Mapping
No custom actions. Create sub-resources:

| Action | Route |
|--------|-------|
| close a card | `POST /cards/:id/closure` |
| archive a card | `POST /cards/:id/archival` |
| watch a board | `POST /boards/:id/watching` |

### Models — State as Records

Instead of booleans, create state records (timestamps + authorship for free):

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
  def close(creator: Current.user) = create_closure!(creator: creator)
  def reopen = closure&.destroy
end

# Querying
Card.joins(:closure)          # closed
Card.where.missing(:closure)  # open
```

### Concerns — Horizontal Behavior

Self-contained (associations + scopes + methods). 50–150 lines. Named for
capabilities, not organization.

```ruby
class Card < ApplicationRecord
  include Assignable, Closeable, Golden, Watchable, Searchable
end
```

### POROs Under Model Namespace

Business logic that doesn't fit a concern:

```ruby
# app/models/event/description.rb
class Event::Description
  def initialize(event) = @event = event
  def to_s = # ...
end
```

### Validations

Minimal on model (data integrity), contextual on form objects (UI flows):

```ruby
# Model — minimal
class User < ApplicationRecord
  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  normalizes :email, with: ->(e) { e.strip.downcase }
end

# Migration — hard constraints
add_index :users, :email, unique: true
add_foreign_key :cards, :boards
```

### Association Design

| Instead of | Prefer | When |
|------------|--------|------|
| `has_and_belongs_to_many` | `has_many :through` | Join model needs attributes or callbacks |
| 1:N | M:N (`has_many :through`) | Relationship may grow |
| `polymorphic: true` | `delegated_type` | Type-safe variants with different schemas |

Always set `:dependent` on `has_many`/`has_one`. Use `:inverse_of` when
Rails can't infer it.

### Rails 7.1+ Patterns

```ruby
# Delegated types
class Message < ApplicationRecord
  delegated_type :messageable, types: %w[Comment Reply Announcement]
end

# store_accessor for JSON/JSONB
class User < ApplicationRecord
  store_accessor :settings, :theme, :notifications_enabled
end
```

### Database

```ruby
# UUIDs (UUIDv7 — time-sortable)
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

### Controllers

Thin. Use concerns for shared behavior:

```ruby
module CardScoped
  extend ActiveSupport::Concern
  included { before_action :set_card }

  private
    def set_card
      @card  = Card.find(params[:card_id])
      @board = @card.board
    end
    def render_card_replacement
      render turbo_stream: turbo_stream.replace(@card)
    end
end

class Cards::ClosuresController < ApplicationController
  include CardScoped
  def create  = @card.close   && render_card_replacement
  def destroy = @card.reopen  && render_card_replacement
end
```

### Current Attributes

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :account, :request_id
  delegate :user, to: :session, allow_nil: true
end
```

### Background Jobs

Thin wrappers — logic lives on the model:

```ruby
module Watchable
  def notify_watchers_later = NotifyWatchersJob.perform_later(self)
  def notify_watchers
    watchers.each { |w| WatcherMailer.notification(w, self).deliver_later }
  end
end

class NotifyWatchersJob < ApplicationJob
  def perform(card) = card.notify_watchers
end
```

### View Helpers

Date formatting via initializer (never `strftime` in views):

```ruby
# config/initializers/date_formats.rb
Date::DATE_FORMATS[:default]    = "%d.%m.%Y"
Time::DATE_FORMATS[:default]    = "%d.%m.%Y %H:%M"
Time::DATE_FORMATS[:time_only]  = "%H:%M"
```

Use `active_link_to` gem for navigation active states.

### Testing

- **Minitest** with **fixtures**, integration tests for controllers
- Test observable behavior, not implementation
- Don't mock what you can test for real
- VCR for external APIs

### Preferred Stack

| Concern | Gem |
|---------|-----|
| Frontend | turbo-rails, stimulus-rails, importmap-rails |
| Assets | propshaft |
| Jobs | Solid Queue |
| Cache/Cable | Solid Cache, Solid Cable |
| Authorization | Pundit |
| Deployment | Kamal + Thruster |

### Gem Selection

1. Can vanilla Rails do this?
2. Is it the app's core concern? If yes, own the code
3. Does it add infrastructure? Database-backed alternatives exist
4. Is the complexity worth it?
5. Is it from someone you trust?

## Deep Reference Files

Read these on demand when the task requires deeper guidance:

- **Design patterns** (form objects, query objects, strategies, etc.): [reference/patterns.md](reference/patterns.md)
- **Anti-patterns and code smells**: [reference/anti-patterns.md](reference/anti-patterns.md), [reference/smells.md](reference/smells.md)
- **Refactoring recipes**: [reference/refactorings/](reference/refactorings/) (002–010)
- **Testing guide**: [reference/shared/testing.md](reference/shared/testing.md)
- **Hotwire** (Turbo + Stimulus): [reference/shared/hotwire.md](reference/shared/hotwire.md)
- **Background jobs + Continuations**: [reference/shared/jobs.md](reference/shared/jobs.md)
- **State machines** (AASM): [reference/shared/state_machines.md](reference/shared/state_machines.md)
- **Callbacks** scoring + extraction: [reference/shared/callbacks.md](reference/shared/callbacks.md)
- **Authorization** (Pundit): [reference/shared/authorization.md](reference/shared/authorization.md)
- **Notifications** (Noticed): [reference/shared/notifications.md](reference/shared/notifications.md)
- **Instrumentation** / EventReporter: [reference/shared/instrumentation.md](reference/shared/instrumentation.md)
- **Components** (Phlex): [reference/shared/components.md](reference/shared/components.md)
- **Serializers** (API JSON): [reference/shared/serializers.md](reference/shared/serializers.md)
- **Architecture layers**: [reference/shared/architecture.md](reference/shared/architecture.md)
- **Gem recommendations**: [reference/toolbelt.md](reference/toolbelt.md)

For frontend patterns (Stimulus controllers, Turbo Frames/Streams), invoke
the relevant `hwc-*` skill alongside this one.
