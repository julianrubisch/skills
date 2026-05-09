# Rails Anti-patterns

> Based on *Rails Antipatterns* by Chad Pytel & Tammer Saleh (thoughtbot).
> Supplements `smells.md` and `review-architecture.md` with anti-patterns
> focused on external service handling, database/migration hygiene,
> session management, and failure handling.

## Voyeuristic Models (Law of Demeter)

> Extends `smells.md § Feature Envy`

**Pattern**: Controllers or views reaching deep into model associations.

**Detection**:
- Chained association access in views: `@invoice.customer.address.street`
- Raw `where`/`order`/`limit` chains in controllers on associations
- Finders defined on wrong model (crossing association boundaries)

**Severity**: Medium

**Solutions**:
```ruby
# Use delegate with prefix
class Invoice < ApplicationRecord
  belongs_to :customer
  delegate :street, :city, :zip, to: :address, prefix: true, allow_nil: true
  # Now: invoice.address_street instead of invoice.customer.address.street
end

# Access finders through association proxy
@memberships = @user.memberships.recently_active  # Good
@memberships = Membership.where(user: @user, active: true).limit(5)  # Bad
```

**Audit Check**: Search views for `@var.association.association.method` (3+ levels).
Search controllers for `.where`/`.order`/`.limit` chains on associations.

## Bloated Sessions

**Pattern**: Storing full objects in session instead of lightweight references.

**Detection**:
- `session[:user] = @user` (full ActiveRecord object)
- `session[:cart] = @cart_items` (collections)
- Session data containing hashes with model attributes

**Severity**: High (causes stale data, serialization issues)

**Solutions**:
```ruby
# Bad: Storing object (stale data, serialization issues)
session[:user] = User.find(params[:id])

# Good: Store only the reference
session[:user_id] = params[:id]

def current_user
  @current_user ||= User.find_by(id: session[:user_id])
end
```

**Audit Check**: Search for `session[` assignments. Flag any storing non-scalar values.

## Fire and Forget (External Service Errors)

**Pattern**: Calling external services without proper exception handling.

**Detection**:
- HTTP calls without `rescue` blocks
- Bare `rescue` or `rescue => e` without specific exception classes
- `rescue nil` on external service calls
- `config.action_mailer.raise_delivery_errors = false` without alternative handling

**Severity**: High

**Solutions**:
```ruby
# Bad: Bare rescue hides real problems
def send_to_api(data)
  ApiClient.post("/webhook", data)
rescue
  nil  # Silently swallows ALL errors
end

# Good: Explicit exception handling with reporting
HTTP_ERRORS = [
  Timeout::Error, Errno::ECONNRESET, Net::HTTPBadResponse,
  Net::HTTPHeaderSyntaxError, Net::ProtocolError, EOFError,
  SocketError, Errno::ECONNREFUSED
].freeze

def send_to_api(data)
  ApiClient.post("/webhook", data)
rescue *HTTP_ERRORS => e
  ErrorTracker.notify(e)  # Sentry, Honeybadger, etc.
  nil
end
```

**Audit Check**: Search for `rescue\s*$` or `rescue\s*=>` (bare rescue),
`rescue nil`. Check `action_mailer` config for suppressed errors.

## Sluggish Services (Missing Timeouts)

**Pattern**: External service calls blocking requests without timeout configuration.

**Detection**:
- HTTP calls without explicit timeout settings
- Synchronous API calls in request cycle that could be backgrounded
- Default `Net::HTTP` timeout (60 seconds) unchanged

**Severity**: High

**Solutions**:
```ruby
# Bad: No timeout — blocks up to 60 seconds
Net::HTTP.get(URI("https://slow-api.example.com/data"))

# Good: Explicit timeouts
uri = URI("https://slow-api.example.com/data")
http = Net::HTTP.new(uri.host, uri.port)
http.open_timeout = 5
http.read_timeout = 5
http.request(Net::HTTP::Get.new(uri))

# Better: Use Faraday with timeouts
conn = Faraday.new(url: "https://api.example.com") do |f|
  f.options.timeout = 5
  f.options.open_timeout = 2
end

# Best: Background non-critical calls
SendWebhookJob.perform_later(data)
```

**Audit Check**: Search for `Net::HTTP`, `Faraday`, `HTTParty`, `RestClient` usage.
Verify timeout configuration. Flag synchronous API calls in controllers that
could be backgrounded.

## Messy Migrations

**Pattern**: Migrations that reference external model code or lack reversibility.

**Detection**:
- Model class references inside migrations (e.g., `User.all.each`)
- Missing `down` method for non-reversible changes
- Migrations modified after being committed
- Missing `reset_column_information` after schema changes in data migrations

**Severity**: Medium

**Solutions**:
```ruby
# Bad: External model dependency — breaks if User model changes
class AddJobsCountToUser < ActiveRecord::Migration[7.1]
  def up
    add_column :users, :jobs_count, :integer, default: 0
    User.all.each { |u| u.update!(jobs_count: u.jobs.size) }
  end
end

# Good: Pure SQL, no external dependencies
class AddJobsCountToUser < ActiveRecord::Migration[7.1]
  def up
    add_column :users, :jobs_count, :integer, default: 0
    execute <<-SQL
      UPDATE users SET jobs_count = (
        SELECT count(*) FROM jobs WHERE jobs.user_id = users.id
      )
    SQL
  end

  def down
    remove_column :users, :jobs_count
  end
end

# If model needed, define inline
class BackfillData < ActiveRecord::Migration[7.1]
  class User < ApplicationRecord
    self.table_name = "users"
  end

  def up
    User.reset_column_information
    # Safe to use inline User class
  end
end
```

**Audit Check**: Search `db/migrate/` for model class names (e.g., `User.`, `Order.`).
Verify each migration has reversible `down` or uses only reversible `change` methods.

## Inaudible Failures (Silent Errors)

> Extends `coding-classic.md` → "Let it crash: use bang methods"

**Pattern**: Code that fails silently — `save` without checking return value, missing preconditions.

**Detection**:
- `save` without checking return value (in jobs, services, rake tasks)
- `update` without checking return value
- Bulk operations without transactions
- Missing fail-fast precondition checks

**Severity**: High

**Solutions**:
```ruby
# Bad: Silent failure
class Ticket < ApplicationRecord
  def self.bulk_change_owner(user)
    all.each do |ticket|
      ticket.owner = user
      ticket.save  # Returns false silently on failure
    end
  end
end

# Good: Fail loudly with transaction
class Ticket < ApplicationRecord
  def self.bulk_change_owner(user)
    transaction do
      all.find_each do |ticket|
        ticket.update!(owner: user)
      end
    end
  end
end

# In background jobs: Use bang methods
class ProcessOrderJob < ApplicationJob
  def perform(order)
    order.process!  # Raises on failure
    order.save!     # Raises on failure
  end
end
```

**Audit Check**: Search for `\.save\b` (without `!`) in models, jobs, rake tasks.
Flag bulk operations without `transaction`. Search for bare `rescue` statements.

## Logic in Views

> See also: `patterns.md § Presenters / Decorators`

**Pattern**: Business logic, queries, or complex conditionals in view templates.

**Detection**:
- Model queries in views: `<% User.where(...) %>`
- Nested `if/else` blocks (> 2 levels)
- Calculations or business logic in templates

**Severity**: Medium

**Solutions**:
```erb
<%# Bad: Logic in view %>
<% if @order.status == 'pending' && @order.created_at > 1.hour.ago %>
  <span class="warning">Processing</span>
<% elsif @order.status == 'shipped' && @order.tracking_number.present? %>
  <span class="success">Shipped</span>
<% end %>

<%# Good: Use presenter %>
<% present(@order) do |o| %>
  <%= o.status_badge %>
<% end %>
```

**Audit Check**: Search `app/views/` for `Model.find`, `Model.where`, `.order`, `.joins`.
Flag views with > 2 levels of conditional nesting.
