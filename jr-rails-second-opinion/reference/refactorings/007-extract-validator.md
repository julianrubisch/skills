# Refactoring: Extract Validator

## Context
Complex or reusable validation logic inline on a model — regex patterns,
multi-step rules, or the same validation appearing on multiple models.
A specialised form of Extract Class targeting `ActiveModel::EachValidator`.

See also: `patterns.md § Rule Objects` for guard-clause-style conditionals
that span multiple fields and don't fit the per-attribute validator pattern.

## Before
```ruby
class Invitation < ApplicationRecord
  EMAIL_REGEX = /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
  validates :recipient_email, presence: true,
                              format: { with: EMAIL_REGEX, message: "is not valid" }
end

# Duplicated in User:
class User < ApplicationRecord
  EMAIL_REGEX = /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i
  validates :email, format: { with: EMAIL_REGEX }
end
```

## After
```ruby
# app/validators/email_address_validator.rb
class EmailAddressValidator < ActiveModel::EachValidator
  EMAIL_REGEX = /\A([^@\s]+)@((?:[-a-z0-9]+\.)+[a-z]{2,})\z/i

  def validate_each(record, attribute, value)
    unless value.match?(EMAIL_REGEX)
      record.errors.add(attribute, "#{value} is not a valid email address")
    end
  end
end

class Invitation < ApplicationRecord
  validates :recipient_email, presence: true, email_address: true
end

class User < ApplicationRecord
  validates :email, email_address: true
end
```

## Why
Single definition of the rule. Reusable across models via the custom validator
key. Follows SRP — the model declares *what* to validate, the validator defines
*how*. Testable in isolation without loading the full model.

## When NOT to apply
- The validation is model-specific and won't be reused — inline is simpler.
- The check spans multiple fields as a guard clause rather than validating one
  attribute — use a Rule Object instead (see `patterns.md § Rule Objects`).

_[Ruby Science →](https://thoughtbot.com/ruby-science/extract-validator.html)_
