# Refactoring: Replace Subclasses with Strategies

## Context
An STI hierarchy where subclasses vary on one behavioral dimension, or where
switching a record's type requires deleting and recreating it. When subclasses
share most state but diverge in methods, composition is cleaner than inheritance.

## Before
```ruby
# STI: all types in one table, `type` column discriminates
class Question < ApplicationRecord; end

class OpenQuestion < Question
  def summary = summarize_open_answers
  def score(value) = value.length > 0 ? 1 : 0
end

class MultipleChoiceQuestion < Question
  def summary = summarize_multiple_choice_answers
  def score(value) = choices.index(value)
end
```

## After
```ruby
# Question owns persistence; strategy owns behavior
class Question < ApplicationRecord
  belongs_to :submittable, polymorphic: true
  delegate :summary, :score, to: :submittable
end

class OpenSubmittable < ApplicationRecord
  def summary = summarize_open_answers
  def score(value) = value.length > 0 ? 1 : 0
end

class MultipleChoiceSubmittable < ApplicationRecord
  has_many :choices
  def summary = summarize_multiple_choice_answers
  def score(value) = choices.index(value)
end
```

## Why
Type switches without deleting the question record — just swap the `submittable`
association. Strategy-specific state lives on dedicated tables (no nullable columns).
Composition is easier to test than STI.

**Signal that you need this:** a subclass overrides the parent's orchestration
method (e.g. `deliver`) just to change one part of the behavior. You're
duplicating orchestration to vary a detail — that's the warning sign.

**Rule of thumb** (Sandi Metz):
- Inherit for "is-a" with a stable type hierarchy.
- Mixin for cross-cutting "acts-as" concerns.
- Compose for "uses-a" — when you need runtime swappability or behaviors
  vary independently.

If you inherit just to reuse code, compose instead.

## When NOT to apply
- Strategies have no state — a plain Ruby strategy object (no AR model) is
  enough; skip the polymorphic association.
- The hierarchy is shallow (2 types) and stable — STI overhead may not be worth
  the composition complexity.
- The strategy needs deep access to the host's internals — you're adding
  indirection for no gain (Feature Envy back at the host).
- "Before performing a large change like this, try to imagine what currently
  difficult changes will be easier to make in the new version." — Ruby Science

_[Ruby Science →](https://thoughtbot.com/ruby-science/replace-subclasses-with-strategies.html)_
