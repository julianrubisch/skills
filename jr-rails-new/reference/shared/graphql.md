# GraphQL Reference

> **Not the preferred API style.** REST with standard Rails controllers is the
> default recommendation (see `coding/api.md`). This reference exists for
> reviewing or maintaining existing GraphQL APIs built with
> [graphql-ruby](https://github.com/rmosolgo/graphql-ruby).

## Schema

```ruby
# app/graphql/my_app_schema.rb
class MyAppSchema < GraphQL::Schema
  query Types::QueryType
  mutation Types::MutationType
  subscription Types::SubscriptionType

  use GraphQL::Dataloader

  max_complexity 300
  max_depth 15
end
```

**Key settings:**
- `use GraphQL::Dataloader` — required to enable batched loading (see below)
- `max_complexity` / `max_depth` — prevent abusive queries

## Type Definitions

```ruby
# app/graphql/types/user_type.rb
module Types
  class UserType < Types::BaseObject
    field :id, ID, null: false
    field :email, String, null: false
    field :name, String, null: true
    field :created_at, GraphQL::Types::ISO8601DateTime, null: false

    field :posts, [Types::PostType], null: true
    field :posts_count, Integer, null: false

    def posts_count
      # Prefer counter_cache or dataloader over object.posts.count (N+1)
      dataloader.with(Sources::CountLoader, Post, :user_id).load(object.id)
    end
  end
end
```

### Query Type

```ruby
# app/graphql/types/query_type.rb
module Types
  class QueryType < Types::BaseObject
    field :user, Types::UserType, null: true do
      argument :id, ID, required: true
    end

    field :users, [Types::UserType], null: true do
      argument :limit, Integer, required: false, default_value: 20
      argument :offset, Integer, required: false, default_value: 0
    end

    def user(id:)
      User.find_by(id: id)
    end

    def users(limit:, offset:)
      User.limit(limit).offset(offset)
    end
  end
end
```

## Mutations

### Base Mutation

```ruby
# app/graphql/mutations/base_mutation.rb
module Mutations
  class BaseMutation < GraphQL::Schema::RelayClassicMutation
    argument_class Types::BaseArgument
    field_class Types::BaseField
    input_object_class Types::BaseInputObject
    object_class Types::BaseObject

    def current_user
      context[:current_user]
    end

    def authenticate!
      raise GraphQL::ExecutionError, "Not authenticated" unless current_user
    end
  end
end
```

### Create Mutation

```ruby
# app/graphql/mutations/create_post.rb
module Mutations
  class CreatePost < BaseMutation
    argument :title, String, required: true
    argument :content, String, required: true
    argument :published, Boolean, required: false

    field :post, Types::PostType, null: true
    field :errors, [String], null: false

    def resolve(title:, content:, published: false)
      authenticate!

      post = current_user.posts.build(
        title: title,
        content: content,
        published: published
      )

      if post.save
        { post: post, errors: [] }
      else
        { post: nil, errors: post.errors.full_messages }
      end
    end
  end
end
```

## DataLoader — Avoiding N+1 Queries

The `GraphQL::Dataloader::Source` `fetch` method receives a batch of keys and
must return results **in the same order**. Use `index_by` for O(n) lookup:

```ruby
# app/graphql/sources/record_loader.rb
class Sources::RecordLoader < GraphQL::Dataloader::Source
  def initialize(model_class, column: :id)
    @model_class = model_class
    @column = column
  end

  def fetch(ids)
    records = @model_class.where(@column => ids).index_by(&@column)
    ids.map { |id| records[id] }
  end
end

# Usage in a type
module Types
  class PostType < Types::BaseObject
    field :author, Types::UserType, null: false

    def author
      dataloader.with(Sources::RecordLoader, User).load(object.user_id)
    end
  end
end
```

## Connection Types (Cursor Pagination)

```ruby
# app/graphql/types/post_connection_type.rb
module Types
  class PostConnectionType < Types::BaseConnection
    edge_type(Types::PostEdgeType)

    field :total_count, Integer, null: false

    def total_count
      object.items.size
    end
  end
end

# Query with filtering and pagination
module Types
  class QueryType < Types::BaseObject
    field :posts, Types::PostConnectionType, null: false, connection: true do
      argument :filter, Types::PostFilterInput, required: false
      argument :order_by, Types::PostOrderEnum, required: false
    end

    def posts(filter: nil, order_by: nil)
      scope = Post.all
      scope = apply_filter(scope, filter) if filter
      scope = apply_order(scope, order_by) if order_by
      scope
    end
  end
end
```

## Authentication & Authorization

### Context Setup

```ruby
# app/controllers/graphql_controller.rb
class GraphqlController < ApplicationController
  def execute
    result = MyAppSchema.execute(
      params[:query],
      variables: ensure_hash(params[:variables]),
      context: {
        current_user: current_user,
        request: request
      },
      operation_name: params[:operationName]
    )
    render json: result
  end

  private

  def current_user
    token = request.headers["Authorization"]&.split(" ")&.last
    User.find_by(api_token: token) if token
  end
end
```

### Field-Level Authorization

```ruby
module Types
  class UserType < Types::BaseObject
    field :email, String, null: false

    field :private_notes, String, null: true

    def private_notes
      return nil unless context[:current_user] == object
      object.private_notes
    end

    def self.authorized?(object, context)
      # Type-level authorization — return false to hide this type entirely
      true
    end
  end
end
```

> For Pundit integration, graphql-ruby ships a built-in
> `GraphQL::Pro::PunditIntegration` (Pro feature). For the open-source gem,
> call Pundit manually in resolvers: `authorize object, :show?`.

## Subscriptions

Subscription logic lives in dedicated classes inheriting
`GraphQL::Schema::Subscription`, not inline on the SubscriptionType:

```ruby
# app/graphql/subscriptions/post_created.rb
class Subscriptions::PostCreated < Subscriptions::BaseSubscription
  argument :user_id, ID, required: false

  field :post, Types::PostType, null: false

  def subscribe(user_id: nil)
    # Return initial value or :no_response
    :no_response
  end

  def update(user_id: nil)
    post = object
    if user_id && post.user_id != user_id
      :no_update  # filter out irrelevant events
    else
      { post: post }
    end
  end
end

# app/graphql/types/subscription_type.rb
module Types
  class SubscriptionType < Types::BaseObject
    field :post_created, subscription: Subscriptions::PostCreated
  end
end
```

Trigger from application code (prefer EventReporter over model callbacks —
see `shared/instrumentation.md`):

```ruby
MyAppSchema.subscriptions.trigger(:post_created, {}, post)
```

## Performance

### Query Complexity Logging

```ruby
# app/graphql/analyzers/log_query_complexity.rb
class LogQueryComplexity < GraphQL::Analysis::QueryComplexity
  def result
    complexity = super
    Rails.logger.info "[GraphQL Complexity] #{complexity}"
  end
end

# Register on schema
class MyAppSchema < GraphQL::Schema
  query_analyzer LogQueryComplexity
  # ...
end
```

### Field-Level Caching

```ruby
module Types
  class PostType < Types::BaseObject
    field :comments_count, Integer, null: false

    def comments_count
      Rails.cache.fetch(["post", object.id, "comments_count"]) do
        object.comments.count
      end
    end
  end
end
```

## Testing

Execute queries against the schema directly. Use Minitest with fixtures:

```ruby
class GraphqlUsersQueryTest < ActiveSupport::TestCase
  USERS_QUERY = <<~GQL
    query($limit: Int) {
      users(limit: $limit) {
        id
        name
        email
      }
    }
  GQL

  test "returns users" do
    result = MyAppSchema.execute(
      USERS_QUERY,
      variables: { limit: 10 },
      context: { current_user: users(:admin) }
    )

    assert_nil result["errors"]
    assert_equal User.count, result["data"]["users"].size
  end

  test "unauthenticated request returns error" do
    result = MyAppSchema.execute(USERS_QUERY, context: { current_user: nil })

    assert result["errors"].present?
  end
end
```

### Mutation Tests

```ruby
class GraphqlCreatePostMutationTest < ActiveSupport::TestCase
  CREATE_POST = <<~GQL
    mutation($input: CreatePostInput!) {
      createPost(input: $input) {
        post { id title }
        errors
      }
    }
  GQL

  test "creates a post" do
    user = users(:david)

    result = MyAppSchema.execute(
      CREATE_POST,
      variables: { input: { title: "New Post", content: "Body" } },
      context: { current_user: user }
    )

    data = result["data"]["createPost"]
    assert_empty data["errors"]
    assert_equal "New Post", data["post"]["title"]
  end

  test "returns validation errors" do
    result = MyAppSchema.execute(
      CREATE_POST,
      variables: { input: { title: "", content: "" } },
      context: { current_user: users(:david) }
    )

    data = result["data"]["createPost"]
    assert_nil data["post"]
    assert data["errors"].any?
  end
end
```

## When to Use GraphQL

GraphQL adds complexity. Prefer REST unless you have a clear need:

- **Multiple mobile/SPA clients** with very different data requirements
- **Deeply nested, graph-like data** where clients need flexible traversal
- **Third-party developer API** where you can't predict query patterns

For internal APIs, admin tools, and simple CRUD — REST is simpler and easier
to cache, test, and secure.

## See Also

- `coding/api.md` — REST API patterns (preferred)
- `shared/serializers.md` — JSON serialization
- `shared/authorization.md` — Pundit policies
- `shared/instrumentation.md` — EventReporter for decoupled triggers
