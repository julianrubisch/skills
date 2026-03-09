# Layered Architecture

## The Four Layers

Rails applications are organized into four architecture layers with unidirectional data flow:

| Layer | Responsibility | Rails Examples |
|-------|----------------|----------------|
| **Presentation** | Handle user interactions, present information | Controllers, Views, Channels, Mailers, GraphQL types |
| **Application** | Organize domain objects for use cases | Service objects, Form objects, Policy objects |
| **Domain** | Entities, rules, invariants, application state | Models, Value objects, Domain events |
| **Infrastructure** | Supporting technologies | Active Record, API clients, File storage |

```
Presentation → Application → Domain → Infrastructure
```

## Rules

### Rule 1: Unidirectional Data Flow

Data flows from top to bottom only. Arrows in the architecture always point downward.

### Rule 2: No Reverse Dependencies

Lower layers must not depend on higher layers. A domain object should never depend on a controller or request object.

**Violations:**

```ruby
# BAD: Model (Domain) depends on Current (Presentation context)
class Post < ApplicationRecord
  def destroy
    self.deleted_by = Current.user  # Hidden dependency!
    super
  end
end
```

**Correct:**

```ruby
# GOOD: Model method accepts explicit parameters
class Post < ApplicationRecord
  def destroy_by(user:)
    self.deleted_by = user
    destroy
  end
end
```

**Exception:** Default arguments and `belongs_to` defaults are acceptable —
the caller can always override them, and they keep the API ergonomic:

```ruby
def close(creator: Current.user)            # Fine — overridable
belongs_to :creator, default: -> { Current.user }  # Fine — convention
```

See `smells.md § Current in Models` for the full signal definition.

### Rule 3: Abstraction Boundaries

Every abstraction layer must belong to a single architecture layer. An abstraction cannot span multiple architecture layers.

**Evaluating abstractions:**
- Does this object depend on objects from a higher layer? → Extract or refactor
- Does this object's responsibility match its architecture layer? → Move if not


## Layer Mapping

### Presentation Layer

**Purpose:** Handle user interactions, present information

**Includes:**
- Controllers (HTTP request/response)
- Views (HTML rendering)
- Channels (WebSocket connections)
- Mailers (email composition)
- API serializers
- Form objects (user input handling — straddle presentation/application boundary)
- Filter objects (request parameter transformation)
- Presenters (view-specific logic — see `patterns.md § Presenters`)
- Components (reusable UI building blocks — see `shared/components.md`)

**Primary concerns:**
- Request parsing and validation
- Authentication
- Response formatting
- User interface logic

### Application Layer

**Purpose:** Organize domain objects for specific use cases

**Includes:**
- Policy objects (authorization)
- DCI contexts (use-case orchestration)

**Primary concerns:**
- Orchestrating domain objects
- Transaction boundaries
- Use-case specific logic

**Warning:** This layer is often overused. Prefer named domain models over
generic service objects — see `patterns.md § Domain Models over Service Objects`.
Don't strip all logic from models into services (anemic models anti-pattern).

### Domain Layer

**Purpose:** Entities, rules, invariants, application state

**Includes:**
- Models (business entities)
- Value objects (immutable concepts)
- Domain events
- Query objects (data retrieval logic)
- Concerns (shared behaviors)

**Primary concerns:**
- Business rules and invariants
- Entity relationships
- Data transformations
- Domain-specific calculations

### Infrastructure Layer

**Purpose:** Supporting technologies

**Includes:**
- Active Record (database access)
- API clients (external services)
- File storage adapters
- Message queue adapters
- Cache implementations
- Active Storage (external object storage)
- Configuration classes (see `shared/configuration.md` for structured config objects)

**Primary concerns:**
- Persistence
- External communication
- Technical implementations
- Configuration management

## Using These Principles

When designing or refactoring code:

1. **Identify the architecture layer** the code belongs to
2. **Check dependencies** — does it depend on higher layers?
3. **Apply specification test** — do tests verify appropriate responsibilities?
4. **Extract if needed** — move code to the correct layer

## The Specification Test

> If the specification of an object describes features beyond the primary
> responsibility of its abstraction layer, those features should be extracted
> into lower layers.

Write test descriptions first (without implementation), then check whether
each test verifies something within the object's layer. Mismatches reveal
extraction candidates.

### Example: Controller

```ruby
class GithubCallbacksControllerTest < ActionDispatch::IntegrationTest
  test "rejects when signature is missing"   # ✓ Authentication — controller concern
  test "rejects when signature is invalid"   # ✓ Authentication — controller concern
  test "handles pull_request event"          # ✗ Business logic — extract to domain
  test "handles issue event"                 # ✗ Business logic — extract to domain
  test "handles missing user"                # ✗ Business logic — extract to domain
end
```

After extraction — controller only tests HTTP concerns, domain model tests
the event handling:

```ruby
# Controller — HTTP layer only
class GithubCallbacksControllerTest < ActionDispatch::IntegrationTest
  test "rejects when signature is missing"
  test "rejects when signature is invalid"
  test "processes valid webhook"
end

# Domain model — business logic
class GithubEventTest < ActiveSupport::TestCase
  test "handles pull_request event"
  test "handles issue event"
  test "handles missing user gracefully"
end
```

### Example: Model

```ruby
class OrderTest < ActiveSupport::TestCase
  test "validates minimum order total"       # ✓ Business rule — domain concern
  test "calculates total with discounts"     # ✓ Domain calculation
  test "sends confirmation email"            # ✗ Presentation — move to controller/form object
  test "syncs to warehouse API"              # ✗ Infrastructure — extract to job
end
```

### Applying the Test

1. **List responsibilities** the code handles
2. **Categorize by layer** using the table in § Layer Mapping
3. **Extract** anything outside the object's primary layer

| Responsibility | Layer | In a controller? |
|----------------|-------|------------------|
| Parse parameters | Presentation | ✓ |
| Authenticate / authorize | Presentation / Application | ✓ |
| Validate inventory, pricing, discounts | Domain | ✗ → model |
| Create records | Domain | ✗ → model or form object |
| Send email | Presentation | ✗ → controller after-action or form object callback |
| Sync to external API | Infrastructure | ✗ → background job |
| Return response | Presentation | ✓ |

### Cost Benefit

Moving logic to lower layers produces faster, simpler, more focused tests:

| Test layer | Speed | Setup | Brittleness |
|------------|-------|-------|-------------|
| Model / unit | Fast | Low | Low |
| Integration / controller | Medium | Medium | Medium |
| System | Slow | High | High |

See also: `shared/testing.md § Per-Layer Test Focus` for what each test
type should and should not verify.

## Common Mistakes

| Mistake | Problem | Solution | See also |
|---------|---------|----------|----------|
| Current in models | Hidden dependency on presentation context | Pass as explicit parameter (default args are fine) | `smells.md § Current in Models` |
| Request in services | Service depends on HTTP layer | Extract value object from request | — |
| Mailer/Notification in callbacks | Model triggers presentation-layer code | Use form object, or move to controller | `smells.md § Callback` |
| SQL in controllers | Presentation doing infrastructure work | Use model scopes or query objects | `review-architecture.md § Heuristics` |
| Business logic in views | Presentation doing domain work | Use presenters or model methods | `anti-patterns.md § Logic in Views` |
