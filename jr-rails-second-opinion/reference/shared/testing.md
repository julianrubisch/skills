# Testing: Shared Reference

Prescriptive guide — how to write tests. Loaded by both coding skills and
review agents. Review-specific concerns (what to flag, severity) live in
review-testing.md.

## Test Suite Quality Characteristics

An effective test suite is:
- **Fast**: Run frequently, quick feedback loop
- **Complete**: All public code paths covered
- **Reliable**: No false positives or intermittent failures
- **Isolated**: Tests run independently, clean up after themselves
- **Maintainable**: Easy to add new tests and modify existing ones
- **Expressive**: Tests serve as documentation

## Stack

- **Minitest** — ships with Rails, lower ceremony than RSpec
- **Fixtures** — not factory_bot
- **VCR** — for external API interactions

## Test Types and When to Use Each

Structure your test suite as a pyramid:
- **Base**: Many fast unit/model tests
- **Middle**: Some integration tests
- **Top**: Few slow feature/system tests

| Type | Use when |
|------|----------|
| Unit (model) | Domain logic, validations, scopes, concerns |
| Integration | Full request/response cycle, controller behavior, auth |
| System | Genuine browser/JS interaction only (drag-drop, Stimulus behavior) |

System tests are expensive — everything that doesn't need a browser should
be an integration or unit test.

## Fixtures

```yaml
# test/fixtures/users.yml
david:
  identity: david
  account: basecamp
  role: admin

jason:
  identity: jason
  account: basecamp
  role: member

# test/fixtures/cards.yml
one:
  title: First Card
  board: main
  creator: david

# Dynamic values via ERB
recent:
  title: Recent Card
  created_at: <%= 1.hour.ago %>

old:
  title: Old Card
  created_at: <%= 1.month.ago %>
```

## Unit Tests

```ruby
class CardTest < ActiveSupport::TestCase
  setup do
    @card = cards(:one)
    @user = users(:david)
  end

  test "closing a card creates a closure" do
    assert_difference -> { Card::Closure.count } do
      @card.close(creator: @user)
    end

    assert @card.closed?
    assert_equal @user, @card.closure.creator
  end

  test "reopening destroys the closure" do
    @card.close(creator: @user)

    assert_difference -> { Card::Closure.count }, -1 do
      @card.reopen
    end

    refute @card.closed?
  end
end
```

### Per-type guidance

| Type | Focus on | Notes |
|------|----------|-------|
| Model | All public methods, edge cases | Every model should have a corresponding test file |
| Controller / Integration | Auth checks, error paths, response formats | Prefer integration tests; use when auth or routing matters |
| View | Conditional rendering, complex logic | Skip if using ViewComponent/Phlex — test those with their own primitives |
| Helper | All public methods | — |
| Mailer | Recipients, subject, body content | — |

### Per-Layer Test Focus

Use this as a layer-violation check: if a test verifies something outside
its layer, that code should be extracted. See `shared/architecture.md §
The Specification Test` for the full technique.

**Controller / Integration tests SHOULD verify:**
- HTTP status codes, redirects, response formats
- Authentication and authorization
- Parameter handling and strong params

**Controller / Integration tests should NOT verify:**
- Business rules, calculations, state transitions
- External service behavior

**Model tests SHOULD verify:**
- Validations, business rules, calculations
- State transitions, scopes, associations with logic

**Model tests should NOT verify:**
- HTTP concerns, notification delivery, external API calls


## Integration Tests

```ruby
class CardsControllerTest < ActionDispatch::IntegrationTest
  setup do
    sign_in users(:david)
  end

  test "closing a card" do
    post card_closure_path(cards(:one))

    assert_response :success
    assert cards(:one).reload.closed?
  end

  test "unauthorized user cannot close card" do
    sign_in users(:guest)

    post card_closure_path(cards(:one))

    assert_response :forbidden
    refute cards(:one).reload.closed?
  end
end
```

## System Tests (Use Sparingly)

**Required Coverage**:
- All critical user flows
- Happy paths for main features
- Key error handling paths

**Audit Checks**:
- [ ] Login/authentication flow tested
- [ ] Main CRUD operations tested
- [ ] Payment flows tested (if applicable)
- [ ] Critical business workflows tested

```ruby
class MessagesTest < ApplicationSystemTestCase
  test "drag and drop card to new column" do
    sign_in users(:david)
    visit board_path(boards(:main))

    card.drag_to find("#column_#{columns(:done).id}")

    assert_selector "#column_#{columns(:done).id} #card_#{cards(:one).id}"
  end
end
```

## Principles

### Four Phase Test Pattern

Every test should follow:

```ruby
test "returns the user's full name" do
  # Setup — use fixtures or build inline
  user = users(:david)

  # Exercise — execute the code being tested
  result = user.full_name

  # Verify — assert outcomes
  assert_equal "David Heinemeier Hansson", result

  # Teardown — handled by the framework (transactions)
end
```

**Audit Check**: Tests should have clear separation between phases.

### Test Observable Behavior, Not Implementation

```ruby
# BAD — testing implementation
test "calls notify method on each watcher" do
  card.expects(:notify).times(3)
  card.close
end

# GOOD — testing behavior
test "watchers receive notifications when card closes" do
  assert_difference -> { Notification.count }, 3 do
    card.close
  end
end
```

### Don't Mock What You Can Test for Real

```ruby
# BAD — over-mocked
test "sending message" do
  room = mock("room")
  room.expects(:messages).returns(stub(create!: mock("message")))
end

# GOOD
test "sending message" do
  sign_in users(:david)
  post room_messages_url(rooms(:watercooler)), params: { message: { body: "Hello" } }

  assert_response :success
  assert Message.exists?(body: "Hello")
end
```

## Time-Dependent Tests

```ruby
test "magic link expires after 15 minutes" do
  magic_link = MagicLink.create!(user: users(:alice))
  travel 16.minutes
  assert magic_link.expired?
end

test "card expires after 30 days" do
  travel_to 31.days.from_now do
    assert cards(:one).expired?
  end
end
```

## Jobs, Email, and Broadcast Assertions

```ruby
test "closing card enqueues notification job" do
  assert_enqueued_with(job: NotifyWatchersJob, args: [cards(:one)]) do
    cards(:one).close
  end
end

test "welcome email is sent on signup" do
  assert_emails 1 do
    Identity.create!(email: "new@example.com")
  end
end

test "message creation broadcasts to room" do
  assert_turbo_stream_broadcasts [rooms(:watercooler), :messages] do
    rooms(:watercooler).messages.create!(body: "Test", creator: users(:david))
  end
end
```

## Edge Cases Checklist

Every public method should be tested against:
- Nil / empty values
- Boundary conditions (0, max, off-by-one)
- Invalid inputs (wrong type, malformed data)
- Error / exception paths
- Authorization failures (wrong role, no session)

## Test Performance

- **Parallel tests** — `parallelize(workers: :number_of_processors)` in
  `test_helper.rb` (Rails default since 6.0)
- **Transactional fixtures** — default; avoid `DatabaseCleaner` unless needed
- **Profile slow tests** — `bin/rails test --verbose` to spot outliers
- **Avoid external calls** — VCR for HTTP, stubs for other I/O

## External APIs

```ruby
test "fetches user data from API" do
  VCR.use_cassette("user_api") do
    user_data = ExternalApi.fetch_user(123)
    assert_equal "John", user_data[:name]
  end
end
```
