# Background Jobs

Reference for ActiveJob patterns with Solid Queue. Jobs should be
idempotent, handle errors gracefully, and pass only primitive arguments
(IDs, strings) — never full objects.

## Basic Job Structure

```ruby
class ProcessOrderJob < ApplicationJob
  queue_as :default

  retry_on ActiveRecord::RecordNotFound, wait: 5.seconds, attempts: 3
  discard_on ActiveJob::DeserializationError

  def perform(order_id)
    order = Order.find(order_id)
    OrderProcessor.new(order).process!
    OrderMailer.confirmation(order).deliver_later
  end
end
```

**Key conventions:**
- Pass IDs, not objects — avoids serialization issues and stale data
- `retry_on` for transient errors, `discard_on` for permanent ones
- Let unexpected errors bubble up to trigger default retry behavior

### Queue Configuration

```ruby
class HighPriorityJob < ApplicationJob
  queue_as :urgent

  # Dynamic queue selection
  queue_as do
    model = arguments.first
    model.premium? ? :urgent : :default
  end
end
```

## Idempotency

Jobs may run more than once. Design for it:

```ruby
class ImportDataJob < ApplicationJob
  def perform(import_id)
    import = Import.find(import_id)
    return if import.completed?

    import.with_lock do
      return if import.completed?  # double-check after lock

      process_import(import)
      import.update!(status: "completed")
    end
  end
end
```

### Database Transactions

```ruby
class UpdateInventoryJob < ApplicationJob
  def perform(product_id, quantity_change)
    ActiveRecord::Base.transaction do
      product = Product.lock.find(product_id)
      product.update_inventory!(quantity_change)

      InventoryAudit.create!(
        product: product,
        change: quantity_change,
        processed_at: Time.current
      )
    end
  end
end
```

## Error Handling

### Retry Strategies

```ruby
class SendEmailJob < ApplicationJob
  retry_on Net::SMTPServerError, wait: :exponentially_longer, attempts: 5
  retry_on Timeout::Error, wait: 1.minute, attempts: 3

  discard_on ActiveJob::DeserializationError do |job, error|
    Rails.logger.error "Failed to deserialize job: #{error.message}"
  end

  def perform(user_id, email_type)
    user = User.find(user_id)
    EmailService.new(user).send_email(email_type)
  end
end
```

### Domain-Specific Error Handling

```ruby
class ProcessPaymentJob < ApplicationJob
  def perform(payment_id)
    payment = Payment.find(payment_id)
    PaymentProcessor.charge!(payment)
  rescue PaymentProcessor::InsufficientFunds
    payment.update!(status: "insufficient_funds")
    PaymentMailer.insufficient_funds(payment).deliver_later
  rescue PaymentProcessor::CardExpired
    payment.update!(status: "card_expired")
    # Don't retry — user must update card
  end
end
```

## Batch Processing

### Fan-Out Pattern

```ruby
class BatchProcessJob < ApplicationJob
  def perform(batch_id)
    batch = Batch.find(batch_id)

    batch.items.find_in_batches(batch_size: 100) do |items|
      items.each { |item| ProcessItemJob.perform_later(item.id) }
      batch.increment!(:processed_count, items.size)
    end
  end
end
```

### Self-Chaining Pattern

```ruby
class LargeDataProcessJob < ApplicationJob
  BATCH_SIZE = 1000

  def perform(dataset_id, offset = 0)
    dataset = Dataset.find(dataset_id)
    batch = dataset.records.offset(offset).limit(BATCH_SIZE)
    return if batch.empty?

    process_batch(batch)
    self.class.perform_later(dataset_id, offset + BATCH_SIZE)
  end
end
```

> For Rails 8.1+, consider **Continuations** instead of self-chaining — see below.

## Continuations (Rails 8.1+)

`ActiveJob::Continuable` breaks long-running jobs into resumable steps.
If a job is interrupted (e.g., during deploy), it resumes from the last
completed step rather than restarting. Especially useful with Kamal, which
gives job containers 30 seconds to shut down by default.

```ruby
class ProcessImportJob < ApplicationJob
  include ActiveJob::Continuable

  def perform(import_id)
    @import = Import.find(import_id)

    # Block format — runs once, skipped on resume
    step :initialize do
      @import.prepare!
    end

    # Step with cursor — progress is saved on interruption
    step :process do |step|
      @import.records.find_each(start: step.cursor) do |record|
        record.process
        step.advance! from: record.id
      end
    end

    # Method format
    step :finalize
  end

  private

  def finalize
    @import.finalize!
  end
end
```

**Step API:**
- `step.cursor` — current progress marker (any serializable value)
- `step.set!(value)` — set cursor to specific value
- `step.advance!` — call `succ` on cursor (integers, dates)
- `step.advance!(from: value)` — advance using non-contiguous IDs
- `step.checkpoint!` — create manual checkpoint without cursor update

**Configuration:**

```ruby
class ProcessImportJob < ApplicationJob
  include ActiveJob::Continuable

  # Max times the job can be resumed (default: unlimited)
  self.max_resumptions = 10

  # Options passed to retry_job on resume (default: { wait: 5.seconds })
  self.resume_options = { wait: 10.seconds }

  # Retry after cursor advance even if step raises (default: true)
  self.resume_errors_after_advancing = true
end
```

**When to use Continuations vs self-chaining:**
- Continuations: multi-phase jobs, cursor-based iteration, deploy-safe processing
- Self-chaining: simple offset-based batching on Rails < 8.1

## Scheduled / Recurring Jobs

### Idempotent Recurring Job

```ruby
class DailyReportJob < ApplicationJob
  def perform(date = Date.current)
    return if Report.exists?(date: date, type: "daily")

    report = Report.create!(
      date: date,
      type: "daily",
      data: generate_report_data(date)
    )

    ReportMailer.daily_report(report).deliver_later
  end

  private

  def generate_report_data(date)
    {
      orders: Order.where(created_at: date.all_day).count,
      revenue: Order.where(created_at: date.all_day).sum(:total),
      new_users: User.where(created_at: date.all_day).count
    }
  end
end
```

### Solid Queue Recurring Schedule

```yaml
# config/recurring.yml
production:
  daily_report:
    class: DailyReportJob
    schedule: every day at 2am
    queue: default

  cleanup_expired_sessions:
    class: CleanupSessionsJob
    schedule: every 6 hours
    queue: low
```

## Queue Configuration (Solid Queue)

```yaml
# config/queue.yml
production:
  dispatchers:
    - polling_interval: 1
      batch_size: 500
  workers:
    - queues: [urgent]
      threads: 3
      processes: 1
    - queues: [default, low]
      threads: 5
      processes: 2
```

**Queue design heuristics:**
- `urgent` — user-facing, latency-sensitive (email confirmations, webhooks)
- `default` — standard processing (order fulfillment, data sync)
- `low` — background maintenance (reports, cleanup, analytics)

## Monitoring

Use `around_perform` for lightweight instrumentation. For the EventReporter
pattern, see `shared/instrumentation.md`.

```ruby
class ApplicationJob < ActiveJob::Base
  around_perform do |job, block|
    start_time = Time.current
    Rails.logger.info "Starting #{job.class.name} with args: #{job.arguments}"

    block.call

    duration = Time.current - start_time
    Rails.logger.info "Completed #{job.class.name} in #{duration.round(2)}s"
  end
end
```

## Testing

Use `ActiveJob::TestHelper` for queue assertions, `perform_now` for
inline execution:

```ruby
class ProcessOrderJobTest < ActiveSupport::TestCase
  include ActiveJob::TestHelper

  test "processes the order" do
    order = orders(:pending)

    ProcessOrderJob.perform_now(order.id)

    assert_equal "processed", order.reload.status
  end

  test "enqueues confirmation email" do
    order = orders(:pending)

    assert_enqueued_with(job: ActionMailer::MailDeliveryJob) do
      ProcessOrderJob.perform_now(order.id)
    end
  end

  test "retries on record not found" do
    assert_enqueued_with(job: ProcessOrderJob) do
      ProcessOrderJob.perform_later(0)
      perform_enqueued_jobs
    end
  end
end
```

### Testing Continuations (Rails 8.1+)

```ruby
class ProcessImportJobTest < ActiveSupport::TestCase
  include ActiveJob::TestHelper

  test "processes all records across steps" do
    import = imports(:with_records)

    ProcessImportJob.perform_now(import.id)

    assert import.reload.finalized?
    assert import.records.all?(&:processed?)
  end

  test "resumes from last completed step after interruption" do
    import = imports(:with_records)

    # Simulate interruption during :process step
    assert_enqueued_with(job: ProcessImportJob) do
      ProcessImportJob.perform_now(import.id)
    end
  end
end
```

## See Also

- `shared/instrumentation.md` — EventReporter for job metrics
- `shared/callbacks.md` — when to use callbacks vs jobs
- `shared/notifications.md` — Noticed for multi-channel delivery from jobs
