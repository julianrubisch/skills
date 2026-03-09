# Testing Review Reference

Diagnostic layer — what to flag, how severe, and why. Prescriptive patterns
(how to write tests correctly) live in shared/testing.md.

## Anti-patterns

### Testing the Wrong Layer

**Problem:** Controller tests verify business logic; model tests verify HTTP behavior.

```ruby
# BAD — controller test checking domain logic
test "applies VIP discount" do
  post :create, params: { items: [...] }
  assert_equal 90, Order.last.total
end

# GOOD — domain logic in model test
test "applies VIP discount" do
  order = Order.new(customer: customers(:vip))
  order.calculate_total
  assert_equal 90, order.total
end
```

### Excessive System Tests

**Problem:** System tests are slow and brittle. A suite dominated by them
gives slow feedback and masks gaps in unit/integration coverage.

**Signal:** `test/system/` has more files than `test/integration/`. Tests that
only assert on response status or database state (no JS interaction needed).

**Fix:** See system test conversion workflow in NOTES.md — triage by churn +
recency, convert candidates to integration/controller tests.

### Factory-Dependent Tests

**Problem:** Factories create objects at runtime, slowing tests and coupling
them to factory definitions. Factory changes break unrelated tests.

**Signal:** `FactoryBot`, `create(:user)`, `build(:card)` in test files.

**Fix:** Fixtures. Loaded once, relationships explicit in YAML, ERB for
dynamic values. See shared/testing.md.

### Mocking Internal Collaborators

**Problem:** Mocking model methods or internal Rails objects creates brittle
tests that break on refactoring without catching real regressions.

**Fix:** Test observable outcomes (record counts, response codes, broadcasts).
Only mock at system boundaries (external APIs, time).

### Missing Tests for Critical Paths

**Signal:** No integration test for authentication, authorization, or payment
flows. Public methods on models with no corresponding test file.

## Heuristics

- If a test doesn't need a browser, it shouldn't be a system test
- Test files should mirror `app/` — `app/models/card.rb` → `test/models/card_test.rb`
- A test relying on mock expectations rather than real assertions is testing implementation
- `assert_difference` is usually better than checking a count before and after
- Fixtures should cover the common cases; create inline only for edge cases

<!-- Add your own testing review heuristics below -->
