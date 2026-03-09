# Refactoring: Replace Conditional with Polymorphism

## Context
A `case` or `if/elsif` chain that branches on an object's type to execute
different behavior. Signals: Case Statement smell, Divergent Change — adding a
new type requires editing the conditional in every place it appears.

## Before
```ruby
class Question < ApplicationRecord
  def summary
    case question_type
    when 'MultipleChoice' then summarize_multiple_choice_answers
    when 'Open'           then summarize_open_answers
    when 'Scale'          then summarize_scale_answers
    end
  end
end
```

## After
```ruby
class MultipleChoiceQuestion < Question
  def summary = summarize_multiple_choice_answers
end

class OpenQuestion < Question
  def summary = summarize_open_answers
end

class ScaleQuestion < Question
  def summary = summarize_scale_answers
end
```

## Why
Each type owns its behavior. Adding a new type is one new class, not N edits
across N conditionals. Follows OCP — open for extension, closed for modification.
Eliminates the Case Statement smell and the Shotgun Surgery risk that follows it.

## When NOT to apply
- You add **behaviors** frequently (not types): polymorphism makes adding new
  behaviors expensive because every subclass must implement them.
- The variation is a one-off with no realistic growth path.
- Variations span **independent axes** (e.g. transport × output format):
  polymorphism creates a class-per-combination explosion. Use Strategy Objects
  instead (see [005](005-replace-subclasses-with-strategies.md) and
  `patterns.md § Strategy Objects`).
- Consider [Replace Subclasses with Strategies](005-replace-subclasses-with-strategies.md)
  if the inheritance hierarchy is already causing STI pain.

_[Ruby Science →](https://thoughtbot.com/ruby-science/replace-conditional-with-polymorphism.html)_
