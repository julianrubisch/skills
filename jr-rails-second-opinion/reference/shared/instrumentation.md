# Instrumentation

## Summary

Instrumentation provides visibility into application behavior through logging, metrics, and tracing. It's an infrastructure concern that should be non-invasive to domain logic.

Rails 8.1+ introduces `ActiveSupport::EventReporter` (`Rails.event`) as a
structured event bus — think "RabbitMQ lite" for in-process event pipelines.
Prefer it over raw `ActiveSupport::Notifications` for application-level events.

## Layer Placement

```
┌─────────────────────────────────────────┐
│ Infrastructure Layer                    │
│  └─ Instrumentation subscribers         │
│  └─ Log formatters                      │
│  └─ Metrics collectors                  │
│  └─ Event pipeline (Rails.event)        │
└─────────────────────────────────────────┘
```

## Key Principles

- **Non-invasive** — domain code shouldn't know about instrumentation
- **Event-driven** — use Rails instrumentation, not inline logging
- **Structured** — use tagged/structured logging
- **Centralized** — configure in initializers, not throughout code

## Event Pipeline: `Rails.event` (Rails 8.1+)

`ActiveSupport::EventReporter` is a structured event bus with automatic source
location tracking, contextual metadata, and filtered subscriptions. It replaces
raw `ActiveSupport::Notifications` for application-level events.

**In-process only** — no external backend required, but also no cross-process
delivery. For distributed event handling, subscribers should enqueue jobs
(Solid Queue / Sidekiq) that do the actual work.

### Emitting Events

```ruby
# Simple event with payload
Rails.event.notify("order.completed", order_id: order.id, total: order.total)

# With event objects for structured events
class OrderCompletedEvent
  attr_reader :order_id, :total

  def initialize(order_id:, total:)
    @order_id = order_id
    @total = total
  end
end

Rails.event.notify(OrderCompletedEvent.new(order_id: order.id, total: order.total))
```

### Subscribing

Subscribers must implement `emit(event)`. Use a filter block to receive only
matching events:

```ruby
# config/initializers/event_subscribers.rb
class OrderEventSubscriber
  def emit(event)
    case event[:name]
    when "order.completed"
      OrderConfirmationMailer.completed(event[:payload][:order_id]).deliver_later
      InventorySyncJob.perform_later(event[:payload][:order_id])
    when "order.cancelled"
      RefundJob.perform_later(event[:payload][:order_id])
    end
  end
end

Rails.event.subscribe(OrderEventSubscriber.new) { |event| event[:name].start_with?("order.") }
```

### Tags and Context

Tags and context add metadata to all events within a block or request scope:

```ruby
# Tags — stack when nested, scoped to block
Rails.event.tagged("graphql") do
  Rails.event.notify("query.executed", query: query_name)
  # Event includes tags: { "graphql" => true }
end

# Context — request/job-scoped, auto-clears after request
Rails.event.set_context(user_id: Current.user&.id, request_id: request.uuid)
# All events in this request include context: { user_id: 42, request_id: "abc" }
```

### Debug Events

Events that should only fire in development/debug mode:

```ruby
Rails.event.debug("cache.detailed_stats", hit_rate: cache.hit_rate)

# Or enable debug temporarily
Rails.event.with_debug do
  Rails.event.debug("query.explain", sql: query.to_sql)
end
```

### Event Structure

Each emitted event is a hash:

```ruby
{
  name: "order.completed",          # String event identifier
  payload: { order_id: 42 },        # Hash (auto-filtered by filter_parameters)
  tags: { "graphql" => true },      # From tagged blocks
  context: { user_id: 1 },          # From set_context
  timestamp: 1234567890123456789,   # Nanosecond precision
  source_location: {                # Automatic caller tracking
    path: "app/models/order.rb",
    lineno: 42,
    label: "complete!"
  }
}
```

### From Model Lifecycle Events

Use `after_commit` to emit events after successful persistence, then let
subscribers handle side effects — keeps models clean:

```ruby
class Order < ApplicationRecord
  after_commit on: :create do
    Rails.event.notify("order.created", id: id, user_id: user_id)
  end

  after_commit on: :update do
    Rails.event.notify("order.updated", id: id, changes: saved_changes.keys)
  end
end
```

## ActiveSupport::Notifications (Rails ≤ 8.0)

For apps not yet on Rails 8.1, use `ActiveSupport::Notifications` directly.
Same subscriber pattern, different API:

### Subscribe to Events

```ruby
# config/initializers/instrumentation.rb
ActiveSupport::Notifications.subscribe("process_action.action_controller") do |event|
  Rails.logger.info({
    event: event.name,
    controller: event.payload[:controller],
    action: event.payload[:action],
    status: event.payload[:status],
    duration_ms: event.duration.round(2)
  }.to_json)
end
```

### Custom Events

```ruby
# Instrument custom operations
class ProcessPayment
  def call(order)
    ActiveSupport::Notifications.instrument(
      "process.payment",
      order_id: order.id,
      amount: order.total
    ) do |payload|
      result = gateway.charge(order)
      payload[:status] = result.success? ? :success : :failure
      result
    end
  end
end

# Subscribe elsewhere
ActiveSupport::Notifications.subscribe("process.payment") do |event|
  Metrics.histogram(
    "payment.duration",
    event.duration,
    tags: { status: event.payload[:status] }
  )
end
```

### LogSubscriber Pattern

```ruby
# app/subscribers/payment_log_subscriber.rb
class PaymentLogSubscriber < ActiveSupport::LogSubscriber
  def process(event)
    info do
      "Payment processed: order=#{event.payload[:order_id]} " \
      "amount=#{event.payload[:amount]} " \
      "duration=#{event.duration.round(2)}ms"
    end
  end

  def refund(event)
    info { "Payment refunded: #{event.payload[:order_id]}" }
  end
end

PaymentLogSubscriber.attach_to :payment
```

## Structured Logging

### Tagged Logging

```ruby
class ApplicationController < ActionController::Base
  around_action :tag_logs

  private

  def tag_logs
    Rails.logger.tagged(
      "request_id:#{request.uuid}",
      "user:#{current_user&.id}"
    ) { yield }
  end
end
```

### JSON Logging

```ruby
# config/environments/production.rb
config.log_formatter = proc do |severity, time, progname, msg|
  {
    severity: severity,
    time: time.iso8601,
    progname: progname,
    message: msg
  }.to_json + "\n"
end
```

## Anti-Patterns

### Logging in Domain Models

```ruby
# BAD: Model knows about logging
class Order < ApplicationRecord
  after_create do
    Rails.logger.info("Order created: #{id}")
  end
end

# GOOD: Emit event, subscribe elsewhere
class Order < ApplicationRecord
  after_commit on: :create do
    Rails.event.notify("order.created", id: id)  # Rails 8.1+
  end
end

# Or use LogSubscriber for custom events
class OrderLogSubscriber < ActiveSupport::LogSubscriber
  def create(event)
    info { "Order created: #{event.payload[:order_id]}" }
  end
end
```

### Metrics Scattered in Code

```ruby
# BAD: Metrics inline with business logic
class ProcessPayment
  def call(order)
    $statsd.increment("payment.attempts")
    result = gateway.charge(order)
    $statsd.increment(result.success? ? "payment.success" : "payment.failure")
    result
  end
end

# GOOD: Separate instrumentation from logic
class ProcessPayment
  def call(order)
    ActiveSupport::Notifications.instrument("process.payment", order_id: order.id) do |payload|
      result = gateway.charge(order)
      payload[:status] = result.success? ? :success : :failure
      result
    end
  end
end

# Metrics collected via subscriber
```

### Verbose Debug Logging

```ruby
# BAD: Excessive logging cluttering code
def process(items)
  Rails.logger.debug("Starting process with #{items.count} items")
  items.each_with_index do |item, i|
    Rails.logger.debug("Processing item #{i}: #{item.inspect}")
  end
  Rails.logger.debug("Process complete")
end

# GOOD: Single instrumented event
def process(items)
  ActiveSupport::Notifications.instrument("process.batch", count: items.count) do
    items.map { |item| transform(item) }
  end
end
```

## Testing

### Testing Events (Rails 8.1+)

```ruby
class OrderEventTest < ActiveSupport::TestCase
  test "emits order.created event on create" do
    events = []
    subscriber = Class.new {
      define_method(:emit) { |event| events << event }
    }.new

    Rails.event.subscribe(subscriber) { |e| e[:name] == "order.created" }

    Order.create!(user: users(:david), total: 1999)

    assert_equal 1, events.size
    assert_equal "order.created", events.first[:name]
  ensure
    Rails.event.unsubscribe(subscriber)
  end
end
```

### Testing Notifications (Rails ≤ 8.0)

```ruby
class ProcessPaymentTest < ActiveSupport::TestCase
  test "instruments the payment operation" do
    events = []
    callback = ->(event) { events << event }

    ActiveSupport::Notifications.subscribed(callback, "process.payment") do
      ProcessPayment.new.call(orders(:pending))
    end

    assert_equal 1, events.size
    assert_equal orders(:pending).id, events.first.payload[:order_id]
    assert_equal :success, events.first.payload[:status]
  end
end
```

### Testing Subscribers

```ruby
class OrderEventSubscriberTest < ActiveSupport::TestCase
  test "enqueues confirmation email on order.completed" do
    subscriber = OrderEventSubscriber.new
    event = { name: "order.completed", payload: { order_id: orders(:completed).id } }

    assert_enqueued_email_with OrderConfirmationMailer, :completed do
      subscriber.emit(event)
    end
  end
end
```

## Performance Considerations

```ruby
# Use lazy evaluation for expensive log messages
Rails.logger.debug { "Expensive: #{expensive_calculation}" }

# Batch metrics
$statsd.batch do |batch|
  batch.increment("foo")
  batch.gauge("bar", 100)
end

# Sample high-volume events
ActiveSupport::Notifications.subscribe("sql.active_record") do |event|
  next unless rand < 0.1  # 10% sample
  # Process event
end
```
