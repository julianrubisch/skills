# Serializers

## Summary

Serializers are specialized presenters for API responses. They form a dedicated abstraction layer between models and the external API contract, providing consistent JSON formatting.

## When to Use

- API responses beyond simple `render json: @model`
- Multiple JSON formats for different contexts (list vs detail, public vs admin)
- Versioned APIs
- Decoupling internal model structure from external API contract

## When NOT to Use

- Simple `render json: @model` covers it
- Internal data transformation
- Hotwire/server-rendered HTML apps (use presenters instead)

## Key Principles

- **Don't override `#as_json`** — couples model to API representation
- **Convention-based lookup** — `PostSerializer` for `Post` model
- **One serializer per context** — different serializers for list vs detail
- **Plain Ruby first** — reach for gems only when the DSL pays for itself

## Implementation

### Plain Ruby Serializer

Uses the same `SimpleDelegator` pattern as presenters, but targets JSON output:

```ruby
class UserSerializer < SimpleDelegator
  def as_json(...)
    {id:, short_name:, email:}
  end

  def short_name
    name.squish.split(/\s/).then do |parts|
      parts[0..-2].map { _1[0] + "." }.join + parts.last
    end
  end
end

class PostSerializer < SimpleDelegator
  def as_json(...)
    {
      id:,
      title:,
      is_draft: draft?,
      user: UserSerializer.new(user)
    }
  end
end
```

### Usage

```ruby
# Single object
render json: PostSerializer.new(post)

# Collection
render json: posts.map { |p| PostSerializer.new(p) }

# With root key
render json: {post: PostSerializer.new(post)}
```

### Convention-Based Lookup

```ruby
class ApplicationController < ActionController::API
  private

  def serialize(obj, with: nil)
    serializer = with || infer_serializer(obj)
    serializer.new(obj)
  end

  def infer_serializer(obj)
    model = obj.respond_to?(:model) ? obj.model : obj.class
    "#{model.name}Serializer".constantize
  end
end

# Usage
render json: serialize(@post)
render json: serialize(@post, with: Post::DetailSerializer)
```

### Context-Specific Serializers

```ruby
# List view (minimal)
class Post::ListSerializer < SimpleDelegator
  def as_json(...)
    {id:, title:, published_at:}
  end
end

# Detail view (full)
class Post::DetailSerializer < SimpleDelegator
  def as_json(...)
    {
      id:, title:, body:, published_at:,
      user: UserSerializer.new(user),
      comments: comments.map { |c| CommentSerializer.new(c) }
    }
  end
end

# Admin view
class Admin::PostSerializer < SimpleDelegator
  include Rails.application.routes.url_helpers

  def as_json(...)
    {
      id:, title:, body:, status:, created_at:, updated_at:,
      user: UserSerializer.new(user),
      edit_url: admin_post_url(self)
    }
  end
end
```

## Alternative: ActiveModel::Serializers

When the DSL pays for itself — associations, conditional attributes, many
serializers with similar structure — `ActiveModel::Serializers` (AMS) reduces
boilerplate. Trade-off: heavier dependency, less control over output shape.

```ruby
class ProductSerializer < ActiveModel::Serializer
  attributes :id, :name, :price, :description, :created_at

  has_many :reviews
  belongs_to :category

  def price
    format("$%.2f", object.price)
  end
end

# Usage — same as plain Ruby serializers
render json: @product                          # auto-discovers ProductSerializer
render json: @products, each_serializer: Product::ListSerializer
```

**When to reach for AMS over plain Ruby:**
- 10+ serializers with similar attribute declarations
- Heavy use of `has_many`/`belongs_to` in JSON responses
- JSON:API format required (`ActiveModel::Serializer::Adapter::JsonApi`)

**When to stay with plain Ruby:**
- Full control over output shape needed
- Few serializers, each with custom logic
- Avoiding the gem dependency

## Anti-Patterns

### Overriding `#as_json` in Models

```ruby
# BAD — couples model to API shape
class Post < ApplicationRecord
  def as_json(options = {})
    super(options.merge(
      only: [:id, :title],
      methods: [:is_draft],
      include: {user: {only: [:id, :name]}}
    ))
  end
end

# GOOD: Use serializer
class PostSerializer < SimpleDelegator
  def as_json(...)
    {id:, title:, is_draft: draft?, user: UserSerializer.new(user)}
  end
end
```

### Multiple Formats Without Serializers

```ruby
# BAD — scattered JSON shape logic
def show
  respond_to do |format|
    format.json { render json: @post.as_json(include: :user) }
  end
end

# GOOD
def show
  respond_to do |format|
    format.json { render json: PostSerializer.new(@post) }
  end
end
```

## Testing

```ruby
class PostSerializerTest < ActiveSupport::TestCase
  def setup
    @post = posts(:draft_post)
    @serialized = PostSerializer.new(@post).as_json
  end

  test "includes expected attributes" do
    assert_equal @post.id, @serialized[:id]
    assert_equal @post.title, @serialized[:title]
    assert_equal true, @serialized[:is_draft]
  end

  test "includes nested user" do
    assert_equal @post.user.id, @serialized[:user][:id]
  end
end
```

## Related Gems

| Gem | Purpose |
|-----|---------|
| [alba](https://github.com/okuramasafumi/alba) | Fast JSON serialization with DSL — consider when plain Ruby serializers get repetitive |
| [typelizer](https://github.com/skryukov/typelizer) | Generate TypeScript types from serializers — useful for SPA frontends |

## See Also

- `coding/api.md` — REST API patterns using serializers
- `shared/graphql.md` — GraphQL types serve a similar role to serializers
