# Refactoring: Extract Inline Query to Named Scope

## Context
Controller actions contain raw `.where(...)` chains that express a named domain concept.
Triggered when the same (or similar) query appears in more than one place, or when
the query intent isn't obvious from reading the code.

## Before
```ruby
# app/controllers/invoices_controller.rb
def index
  @invoices = Invoice.where("due_date < ?", Date.today)
                     .where(paid: false)
                     .order(:due_date)
end

# app/jobs/invoice_reminder_job.rb
def perform
  Invoice.where("due_date < ?", Date.today).where(paid: false).each do |invoice|
    InvoiceMailer.reminder(invoice).deliver_later
  end
end
```

## After
```ruby
# app/models/invoice.rb
class Invoice < ApplicationRecord
  scope :overdue, -> { where("due_date < ?", Date.today).where(paid: false).order(:due_date) }
end

# app/controllers/invoices_controller.rb
def index
  @invoices = Invoice.overdue.page(params[:page])
end

# app/jobs/invoice_reminder_job.rb
def perform
  Invoice.overdue.each do |invoice|
    InvoiceMailer.reminder(invoice).deliver_later
  end
end
```

## Why
The domain concept ("overdue invoice") now has a single, named definition on the model
where it belongs. Controllers and jobs read like intent, not implementation. Changing
the overdue rule is a one-line edit in one place.

## When NOT to apply
- The query is truly one-off and unlikely to be reused
- The conditions come from user input (filtering/search) — use a query object instead
