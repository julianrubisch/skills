# Refactoring: Replace Conditional with Null Object

## Context
The same nil check repeated across multiple call sites — `x.try(:method)`,
`x&.method || default`, `return unless x`. The absent value is a concept worth
naming as an explicit object.

## Before
```ruby
def most_recent_answer_text
  answers.most_recent.try(:text) || Answer::MISSING_TEXT
end

def display_answer
  return "(no response)" unless answers.most_recent
  answers.most_recent.formatted_text
end
```

## After
```ruby
class NullAnswer
  def text           = "No response"
  def formatted_text = "(no response)"
end

class Answer < ApplicationRecord
  def self.most_recent
    order(:created_at).last || NullAnswer.new
  end
end

# Callers no longer check for nil
def most_recent_answer_text = answers.most_recent.text
def display_answer          = answers.most_recent.formatted_text
```

## Why
Absence is modelled explicitly rather than scattered as nil guards. Follows
Tell, Don't Ask — callers send messages without checking state first. One place
to change the "nothing here" representation.

## When NOT to apply
- Nil is genuinely meaningful (an error state, not just absence).
- The null object would need to duplicate most of the real class's API —
  the abstraction may not be worth it.
- Consumers need to distinguish real from null (e.g. for `persisted?` checks) —
  adds conditional logic back in views.

Don't introduce a null object until the same nil check appears in three or
more places.

_[Ruby Science →](https://thoughtbot.com/ruby-science/replace-conditional-with-null-object.html)_
