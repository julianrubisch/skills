# Authorization

## Summary

Authorization determines what authenticated users can do. It belongs in the application layer as policy objects, enforced at presentation layer entry points, never leaking into domain models.

## Layer Placement

```
┌─────────────────────────────────────────┐
│ Presentation Layer                      │
│  └─ Enforcement (authorize calls)       │
├─────────────────────────────────────────┤
│ Application Layer                       │
│  └─ Policy Objects (rules live here)    │
├─────────────────────────────────────────┤
│ Domain Layer                            │
│  └─ Models operate in authorized context│
└─────────────────────────────────────────┘
```

## Key Principles

- **Single enforcement point** — authorize in controllers, nowhere else
- **Policies are application layer** — bridge between presentation and domain
- **Models are agnostic** — domain doesn't know about permissions
- **Scoping over checking** — load only accessible records when possible
- **Minimal rules** — `show?` for visibility, `manage?` for modifications
- **Deny by default** — ApplicationPolicy returns `false` for everything

## Implementation with Pundit

### Basic Policy

```ruby
class PostPolicy < ApplicationPolicy
  def show?
    true  # Public
  end

  def update?
    owner? || admin?
  end

  def destroy?
    admin?
  end

  private

  def owner?
    record.author_id == user.id
  end

  def admin?
    user.admin?
  end
end
```

### Rule Design

Keep rules minimal and consistent:

| Rule | Purpose |
|------|---------|
| `show?` | Visibility — who can see this resource |
| `manage?` | Management — fallback for update?, destroy?, etc. |
| `create?` | Creation — often delegates to parent's manage? |
| `index?` | Listing — often same as show? or delegates to parent |

**Custom rules** are appropriate for domain operations (`transfer?`, `cancel?`) and parent policy groupings (`manage_billing?`). If a custom rule maps to a full controller with its own views, consider a dedicated resource instead.

### Controller Enforcement

```ruby
class PostsController < ApplicationController
  def show
    @post = Post.find(params[:id])
    authorize @post
  end

  def update
    @post = Post.find(params[:id])
    authorize @post

    if @post.update(post_params)
      redirect_to @post
    else
      render :edit, status: :unprocessable_entity
    end
  end
end
```

### Scoping-Based Authorization

Prefer loading only accessible records:

```ruby
class PostsController < ApplicationController
  def index
    @posts = policy_scope(Post)
  end

  def destroy
    @post = policy_scope(Post).find(params[:id])
    authorize @post
    @post.destroy!
    redirect_to posts_path
  end
end

class PostPolicy < ApplicationPolicy
  class Scope < ApplicationPolicy::Scope
    def resolve
      if user.admin?
        scope.all
      else
        scope.where(author: user).or(scope.published)
      end
    end
  end
end
```

### View Authorization

```erb
<% @posts.each do |post| %>
  <article>
    <h2><%= post.title %></h2>

    <% if policy(post).update? %>
      <%= link_to "Edit", edit_post_path(post) %>
    <% end %>

    <% if policy(post).destroy? %>
      <%= button_to "Delete", post, method: :delete %>
    <% end %>
  </article>
<% end %>
```

### Attribute-Based Access Control (ABAC)

```ruby
class DocumentPolicy < ApplicationPolicy
  def view?
    return true if record.public?
    return true if user.department == record.department
    return true if record.shared_with?(user)
    false
  end

  def edit?
    return false if record.locked?
    return true if record.owner == user
    return true if user.manager_of?(record.owner)
    false
  end
end
```

## Error Handling

Return 404 for visibility failures (don't leak resource existence), 403 for permission failures:

```ruby
class ApplicationController < ActionController::Base
  include Pundit::Authorization

  rescue_from Pundit::NotAuthorizedError, with: :handle_unauthorized

  private

  def handle_unauthorized(exception)
    if exception.query == "show?"
      raise ActiveRecord::RecordNotFound  # 404 — don't leak existence
    else
      head :forbidden  # 403 — record exists but action denied
    end
  end
end
```

## N+1 Authorization

```erb
<%# BAD: N policy checks %>
<% @posts.each do |post| %>
  <% if policy(post).publish? %>
    <%= button_to "Publish", ... %>
  <% end %>
<% end %>
```

Solutions:

1. **Preload required data** for policy checks
2. **Cache authorization** results
3. **Use scoping** instead of per-record checks

```ruby
# Preload in controller
class PostsController < ApplicationController
  def index
    @posts = Post.includes(:author).all
    @editable_ids = policy_scope(Post).pluck(:id).to_set
  end
end

# In view
<% if @editable_ids.include?(post.id) %>
```

## Anti-Patterns

### Authorization in Models

```ruby
# BAD: Domain layer knows about permissions
class Post < ApplicationRecord
  def editable_by?(user)
    author == user || user.admin?
  end
end

# GOOD: Keep in policy
class PostPolicy < ApplicationPolicy
  def update?
    owner? || admin?
  end
end
```

### Multiple Enforcement Points

```ruby
# BAD: Authorization in controller AND service
class PostsController < ApplicationController
  def publish
    authorize @post, :publish?
    PublishService.call(@post, current_user)  # Checks again!
  end
end

class PublishService
  def call(post, user)
    raise unless PostPolicy.new(user, post).publish?  # Duplicate!
    post.publish!
  end
end

# GOOD: Single enforcement point
class PostsController < ApplicationController
  def publish
    authorize @post, :publish?
    @post.publish!
  end
end
```

### Implicit Authorization

```ruby
# BAD: Authorization hidden in scope
class PostsController < ApplicationController
  def index
    @posts = current_user.posts  # Implicitly authorized
  end
end

# GOOD: Explicit authorization
class PostsController < ApplicationController
  def index
    @posts = policy_scope(Post)
  end
end
```

## Testing

### Policy Unit Tests

```ruby
class PostPolicyTest < ActiveSupport::TestCase
  def setup
    @post = posts(:draft_post)
  end

  test "allows owner to update" do
    policy = PostPolicy.new(users(:author), @post)
    assert policy.update?
  end

  test "allows admin to update" do
    policy = PostPolicy.new(users(:admin), @post)
    assert policy.update?
  end

  test "denies others from updating" do
    policy = PostPolicy.new(users(:regular), @post)
    assert_not policy.update?
  end

  test "only admin can destroy" do
    assert PostPolicy.new(users(:admin), @post).destroy?
    assert_not PostPolicy.new(users(:author), @post).destroy?
  end
end
```

### Scope Tests

```ruby
class PostPolicyScopeTest < ActiveSupport::TestCase
  test "admin sees all posts" do
    scope = PostPolicy::Scope.new(users(:admin), Post.all).resolve
    assert_equal Post.count, scope.count
  end

  test "regular user sees own and published posts" do
    user = users(:regular)
    scope = PostPolicy::Scope.new(user, Post.all).resolve
    assert scope.all? { |p| p.author == user || p.published? }
  end
end
```

### Controller Authorization Tests

```ruby
class PostsControllerTest < ActionDispatch::IntegrationTest
  test "unauthorized user cannot destroy post" do
    sign_in users(:regular)
    post = posts(:draft_post)

    assert_no_difference "Post.count" do
      delete post_path(post)
    end

    assert_response :forbidden
  end

  test "admin can destroy post" do
    sign_in users(:admin)

    assert_difference "Post.count", -1 do
      delete post_path(posts(:draft_post))
    end

    assert_redirected_to posts_path
  end
end
```

## Related Gems

| Gem | Purpose |
|-----|---------|
| [pundit](https://github.com/varvet/pundit) | Policy-based authorization with convention over configuration |
