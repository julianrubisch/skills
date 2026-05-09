# Current Attributes

## Summary

`Current` is Rails' built-in thread-local storage for request-scoped context like current user, tenant, or request metadata. It enables implicit context passing but must be used carefully to avoid coupling layers.

## Layer Placement

```
┌─────────────────────────────────────────┐
│ Presentation Layer                      │
│  └─ Sets Current values in controllers  │
├─────────────────────────────────────────┤
│ Application Layer                       │
│  └─ May read Current (sparingly)        │
├─────────────────────────────────────────┤
│ Domain Layer                            │
│  └─ Should NOT access Current           │
└─────────────────────────────────────────┘
```

## Key Principles

- **Set at entry points** — controllers, jobs, middleware
- **Read sparingly** — prefer explicit parameters
- **Avoid in model method bodies** — domain layer should be Current-agnostic;
  default arguments and `belongs_to` defaults are acceptable (see below)
- **Reset automatically** — Rails clears between requests

## Implementation

### Basic Current Class

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :user, :request_id, :tenant

  resets { Time.zone = nil }

  def user=(user)
    super
    Time.zone = user&.time_zone
  end
end
```

### Setting in Controllers

```ruby
class ApplicationController < ActionController::Base
  before_action :set_current_attributes

  private

  def set_current_attributes
    Current.user = current_user
    Current.request_id = request.uuid
  end
end
```

### Multi-Tenancy with Current

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :tenant

  def tenant=(tenant)
    super
    ActsAsTenant.current_tenant = tenant if defined?(ActsAsTenant)
  end
end

class ApplicationController < ActionController::Base
  before_action :set_tenant

  private

  def set_tenant
    Current.tenant = Tenant.find_by!(subdomain: request.subdomain)
  end
end
```

### In Background Jobs

```ruby
class ApplicationJob < ActiveJob::Base
  around_perform do |job, block|
    Current.set(
      user: job.arguments.first[:current_user],
      tenant: job.arguments.first[:current_tenant]
    ) do
      block.call
    end
  end
end

# Enqueue with context
ProcessOrderJob.perform_later(
  order: order,
  current_user: Current.user,
  current_tenant: Current.tenant
)
```

## Acceptable Uses

### Audit Logging

```ruby
class ApplicationRecord < ActiveRecord::Base
  before_save :set_audit_user

  private

  def set_audit_user
    self.updated_by = Current.user if respond_to?(:updated_by=)
    self.created_by = Current.user if respond_to?(:created_by=) && new_record?
  end
end
```

### Request Logging

```ruby
class ApplicationController < ActionController::Base
  around_action :tag_logs

  private

  def tag_logs
    Rails.logger.tagged(
      "user:#{Current.user&.id}",
      "request:#{Current.request_id}"
    ) { yield }
  end
end
```

### Time Zone

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :user

  def user=(user)
    super
    Time.zone = user&.time_zone || "UTC"
  end
end
```

## Anti-Patterns

### Current in Model Method Bodies

```ruby
# BAD: Buried, non-overridable dependency
class Post < ApplicationRecord
  def process!
    log_action(Current.user)   # caller can't inject a different user
  end

  before_validation :set_author, on: :create

  def set_author
    self.author = Current.user  # silent nil in jobs/seeds!
  end
end

# GOOD: Default argument — caller can always override
class Post < ApplicationRecord
  def close(creator: Current.user)
    create_closure!(creator:)
  end
end

# GOOD: belongs_to convenience default
class Post < ApplicationRecord
  belongs_to :author, class_name: "User", default: -> { Current.user }
end

# GOOD: Explicit in controller
class PostsController < ApplicationController
  def create
    @post = current_user.posts.build(post_params)
    # ...
  end
end
```

**Rule of thumb:** `Current.*` inside a method body with no way for the caller
to override it is a smell. Default arguments and `belongs_to` defaults are fine
because the caller retains control.

### Current for Business Logic

```ruby
# BAD: Business decisions based on Current
class Post < ApplicationRecord
  def can_publish?
    Current.user&.admin? || author == Current.user
  end
end

# GOOD: Pass context explicitly or use policies
class PostPolicy < ApplicationPolicy
  def publish?
    user.admin? || record.author == user
  end
end
```


### Testing Difficulties

```ruby
# BAD: Tests must set Current
class PostTest < ActiveSupport::TestCase
  setup { Current.user = users(:david) }  # Setup for every test!

  test "sets author" do
    post = Post.create!(title: "Test")
    assert_equal Current.user, post.author
  end
end

# GOOD: Explicit association, no Current needed
class PostTest < ActiveSupport::TestCase
  test "belongs to author" do
    user = users(:david)
    post = user.posts.create!(title: "Test")
    assert_equal user, post.author
  end
end
```

## Where Current IS Appropriate

| Use Case | Appropriate? | Notes |
|----------|--------------|-------|
| Audit trails | ✅ | Infrastructure concern |
| Request logging | ✅ | Infrastructure concern |
| Time zone | ✅ | Presentation concern |
| Locale | ✅ | Presentation concern |
| Multi-tenancy | ⚠️ | Use dedicated gems |
| Authorization | ❌ | Use policies |
| Business logic | ❌ | Pass explicitly |
| Model defaults | ⚠️ | OK as default args / `belongs_to` defaults; not in method bodies |

## Testing with Current

```ruby
# Use Current.set block to ensure cleanup
class AuditLoggingTest < ActiveSupport::TestCase
  test "logs with user context" do
    Current.set(user: users(:david)) do
      # Current.user available here, reset automatically after block
      post = Post.create!(title: "Test", author: users(:david))
      assert_equal users(:david), post.updated_by
    end
  end
end

# Prefer explicit params — no Current needed
class PublishPostTest < ActiveSupport::TestCase
  test "publishes when authorized" do
    user = users(:david)
    post = posts(:draft_post)

    result = PublishPost.new.call(post, by: user)
    assert result.success?
  end
end
```

## Current vs Explicit Parameters

| Approach | Pros | Cons |
|----------|------|------|
| Current | Less boilerplate, automatic propagation | Hidden dependencies, testing complexity |
| Explicit | Clear dependencies, easy testing | More verbose, must pass through layers |

**Recommendation**: Default to explicit parameters. Use Current only for cross-cutting infrastructure concerns (logging, audit, time zone).
