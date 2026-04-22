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

## Where to Put the Extracted Behavior

When the conditional is a type-check (`is_a?`, `kind_of?`, `case obj; when Type`)
that crosses layer boundaries, picking the right destination matters more than
the polymorphism itself:

| The differing behavior needs... | Extract to | Example |
|---------------------------------|------------|---------|
| Nothing beyond the model's own data | Concern included by each type | `#summarize`, `#eligible?` |
| Route helpers, URL generation, formatting | Presenter on each type | `#path`, `#badge_label` |
| Markup / HTML rendering | Component or partial per type | `TypeComponent` |
| A persisted attribute drives the fork (not the class) | `delegated_type` or state machine | `commentable_type`, `status` |

See `shared/architecture.md § Rule 2` for the abstraction ladder that explains
why each destination is the right rung.

### Shape A: Controller/view branches on type → Presenter

```ruby
# Before — view branches on type for URL + label
<% case item %>
<% when Project %>
  <%= link_to item.name, project_path(item), class: "badge-blue" %>
<% when Task %>
  <%= link_to item.title, project_task_path(item.project, item), class: "badge-green" %>
<% end %>

# After — each type's presenter owns its URL/label
<%= link_to item.presenter.search_label,
      item.presenter.search_path,
      class: item.presenter.badge_class %>
```

Presenters live in `app/presenters/`, include `Rails.application.routes.url_helpers`,
and handle view/URL concerns that don't belong on the model.
See `patterns.md § Presenters`.

### Shape B: Model branches on type column → Concern or delegated_type

```ruby
# Before — model branches on category for message + icon
def message
  case category
  when "mention"  then "#{actor.name} mentioned you"
  when "assigned" then "#{actor.name} assigned you"
  end
end

# After — delegated_type, each type owns its behavior
delegated_type :notifiable, types: %w[MentionNotification AssignmentNotification]
delegate :message, :icon, to: :notifiable
```

Use `delegated_type` when the fork is persisted and each type has its own
schema. Use a shared concern when the types share a schema but differ in
behavior.

## When NOT to Apply

- You add **behaviors** frequently (not types): polymorphism makes adding new
  behaviors expensive because every subclass must implement them.
- The variation is a one-off with no realistic growth path.
- The `case` is purely a data lookup (label, icon) — use a constant hash.
- Variations span **independent axes** (e.g. transport × output format):
  polymorphism creates a class-per-combination explosion. Use Strategy Objects
  instead (see [005](005-replace-subclasses-with-strategies.md) and
  `patterns.md § Strategy Objects`).
- You're inside `app/models/` branching on your *own* attributes — that's the
  model doing its job, not a mis-tiered method.
- Consider [Replace Subclasses with Strategies](005-replace-subclasses-with-strategies.md)
  if the inheritance hierarchy is already causing STI pain.

_[Ruby Science →](https://thoughtbot.com/ruby-science/replace-conditional-with-polymorphism.html)_
