# Architecture Review Reference

## Patterns

### Domain Logic in Models

Models own their domain rules. The test: does this method answer a question
or enforce a rule about the model itself? If yes, it belongs here.

```ruby
# GOOD — model knows its own rules
class Order < ApplicationRecord
  def total
    items.sum(&:subtotal).tap { |t| apply_vip_discount(t) if customer.vip? }
  end
end
```

### DCI (Data, Context, Interaction) for Complex Behavioral Contexts

When the same model needs different behavior in different use cases, DCI
injects role modules at runtime via a Context object. Avoids fat models
without resorting to service objects. Best suited to genuinely use-case-specific
behavior, not reusable logic.

```ruby
# Role: behavior injected at runtime
module TransferRole
  def transfer(amount, to:)
    self.balance -= amount
    to.balance += amount
  end
end

# Context: assembles objects + roles, runs the interaction
class Transfer
  def initialize(source, destination, amount)
    source.extend(TransferRole)
    @source = source
    @destination = destination
    @amount = amount
  end

  def call
    @source.transfer(@amount, to: @destination)
  end
end
```

**When to reach for DCI:** the same model needs different collaborators and
behaviors in 2+ distinct use cases.

<!-- Add further preferred patterns here: form objects, query objects, value objects, etc. -->

## Anti-patterns

### Service Objects

**Problem:** Service objects don't communicate intent — `CreateUser`,
`ProcessPayment` are procedures dressed up as classes. They accumulate like
fat models did, with no design pressure on what belongs inside. The name
tells you *what happens*, not *what domain concept this represents*.

<!-- Add more detail here on what to use instead — DCI, rich models, named domain objects -->

### Business Logic in Jobs

**Problem:** Jobs accumulate domain logic because they're "close" to the
models they operate on. Makes logic hard to test without a queue, and creates
idempotency challenges (Sidekiq doesn't guarantee exactly-once execution).

**Signal:** Job `perform` is more than ~5 lines, contains conditionals or
multi-step domain operations.

**Fix:** Job calls a model method or DCI context; logic lives there.

### Fat Models

**Problem:** All logic pulled into the model because "skinny controller, fat model"
was taken too literally. The model ends up with dozens of unrelated responsibilities.

**Signal:** Model over ~200 lines, methods that reference other models
extensively (feature envy), callbacks that trigger side effects outside the
model's own state.

**Fix:** Extract to value objects, query objects, or DCI contexts. Keep
orchestration (mailers, jobs, external calls) out of models entirely.

## Heuristics

- If you can't name a class without "and" or "Manager", it has too many responsibilities
- Jobs should be thin dispatchers — logic belongs in the domain layer
- Ask of every callback: own state change, or side effect? Side effects belong in the caller
- Prefer named domain objects (nouns) over procedural service objects (verb + "Service")

<!-- Add your own architectural rules below -->
