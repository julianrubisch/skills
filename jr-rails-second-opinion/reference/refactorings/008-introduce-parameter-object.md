# Refactoring: Introduce Parameter Object

## Context
A method that takes 3+ related arguments that always appear together. The
argument group is a data clump — it wants to be named and encapsulated.

See also: `patterns.md § Value Objects` — if equality and immutability matter,
go straight to a Value Object. A Parameter Object is the simpler stepping stone
when you just need to group and name data.

## Before
```ruby
def completion_notification(first_name, last_name, email)
  UserMailer.notify(
    name:  "#{first_name} #{last_name}",
    email: email
  ).deliver_later
end

def welcome_message(first_name, last_name, email)
  "Welcome, #{first_name} #{last_name}! Check #{email} for details."
end
```

## After
```ruby
# Ruby 3.2+ Data class — immutable, value-equality, no boilerplate
Recipient = Data.define(:first_name, :last_name, :email) do
  def full_name = "#{first_name} #{last_name}"
end

def completion_notification(recipient)
  UserMailer.notify(name: recipient.full_name, email: recipient.email).deliver_later
end

def welcome_message(recipient)
  "Welcome, #{recipient.full_name}! Check #{recipient.email} for details."
end
```

## Why
The related data has a name. Shared derived behavior (like `full_name`) lives
in one place. Method signatures stay stable when new fields join the group.
Callers construct a `Recipient` once rather than threading three arguments
through multiple layers.

## When NOT to apply
- The arguments aren't genuinely related — grouping them creates a false object.
- Only one method uses the group — wait until a second call site appears before
  extracting.

## AR integration
When the parameter group corresponds to denormalized columns on a single table,
use `composed_of` to map them to the value object without a migration — see
`patterns.md § Value Objects`.

_[Ruby Science →](https://thoughtbot.com/ruby-science/introduce-parameter-object.html)_
