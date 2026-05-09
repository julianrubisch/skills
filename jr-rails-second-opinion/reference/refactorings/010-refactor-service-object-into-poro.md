# Refactoring: Rename Service Object to Domain Model

## Context

A class named `*Service`, `*Manager`, `*Handler`, or `*Processor` that
encapsulates a business operation. The procedural name obscures the domain
concept it represents and provides no design pressure on scope — anything
can be added to a "service."

**Detection signals:**
- `app/services/` directory
- Classes ending in `Service`, `Manager`, `Handler`, `Processor`, `Creator`
- A single `.call` class method with a bag of keyword arguments
- No validations, no error objects — just `raise` or `Result` structs

## Before

```ruby
# app/services/user_registration_service.rb
class UserRegistrationService
  def self.call(email:, password:, company_name:)
    user = User.create!(email:, password:)
    Company.create!(name: company_name, owner: user)
    RegistrationMailer.welcome(user).deliver_later
    user
  end
end

# Controller
def create
  result = UserRegistrationService.call(**registration_params)
  redirect_to dashboard_path
rescue ActiveRecord::RecordInvalid
  render :new
end
```

## Steps

### 1. Identify the Domain Concept

Ask: what *is* this thing? Not what it *does*.

| Instead of | Use |
|------------|-----|
| `UserRegistrationService` | `Registration` |
| `PaymentProcessor` | `Payment` |
| `NotificationService` | `Notification` or `NotificationDelivery` |
| `OrderCreator` | `Order` or `OrderPlacement` |

### 2. Create the Domain Model with ActiveModel::Model

Move arguments to `attr_accessor` or `attribute`. Add validations that were
previously missing or buried in the controller.

```ruby
# app/models/registration.rb
class Registration
  include ActiveModel::Model

  attr_accessor :email, :password, :company_name

  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validates :company_name, presence: true
end
```

### 3. Replace `.call` with a Domain Method

Use `.save` (for ActiveRecord-like semantics) or a domain verb (`.complete`,
`.deliver`, `.place`). Wrap in a transaction if touching multiple models.

```ruby
class Registration
  include ActiveModel::Model

  attr_accessor :email, :password, :company_name

  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validates :company_name, presence: true

  def save
    return false unless valid?

    ApplicationRecord.transaction do
      create_user
      create_company
      send_welcome_email
    end

    true
  end

  private

  def create_user
    @user = User.create!(email:, password:)
  end

  def create_company
    Company.create!(name: company_name, owner: @user)
  end

  def send_welcome_email
    RegistrationMailer.welcome(@user).deliver_later
  end
end
```

### 4. Update the Controller

Same shape as any model-backed controller. `form_with` integration comes free.

```ruby
# After
def create
  @registration = Registration.new(registration_params)

  if @registration.save
    redirect_to dashboard_path
  else
    render :new
  end
end
```

### 5. Delete the Service File

Remove `app/services/<old_name>.rb`. If the directory is now empty, delete it.

## After (complete)

A domain model in `app/models/` with validations, form integration, and a
clear name. The controller looks identical to any scaffold controller.

## Why

- The domain concept has a name (`Registration`, not `UserRegistrationService`).
- Validations, error handling, and form integration come free via `ActiveModel::Model`.
- Natural design pressure — only registration-related logic belongs here.
- Controllers stay thin and uniform.

## When NOT to Apply

- The "service" is a thin wrapper around a single model method — just call
  the method directly.
- The operation is purely infrastructural (e.g., `S3Uploader`) with no domain
  concept behind it.
- The orchestration is complex enough to warrant `ApplicationForm` with
  `submit!` / `with_transaction` / `after_save` callbacks — use a Form Object instead.

See also: `patterns.md § Domain Models over Service Objects`,
`review-architecture.md § Anti-patterns > Service Objects`
