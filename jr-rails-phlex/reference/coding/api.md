# API Development

Domain-specific guidelines for building Rails APIs. Style-agnostic — referenced
by `coding-classic.md`, `coding-phlex.md`, and other coding skills.

## Base Controller

```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ActionController::API
  include ActionController::HttpAuthentication::Token::ControllerMethods

  before_action :authenticate

  rescue_from ActiveRecord::RecordNotFound, with: :render_not_found
  rescue_from ActiveRecord::RecordInvalid, with: :render_unprocessable
  rescue_from ActionController::ParameterMissing, with: :render_bad_request

  private

  def authenticate
    authenticate_or_request_with_http_token do |token, _options|
      @current_user = User.find_by(api_token: token)
    end
  end

  def render_not_found(exception)
    render json: { error: exception.message }, status: :not_found
  end

  def render_unprocessable(exception)
    render json: { errors: exception.record.errors }, status: :unprocessable_entity
  end

  def render_bad_request(exception)
    render json: { error: exception.message }, status: :bad_request
  end
end
```

**Key decisions:**
- Inherit `ActionController::API` not `ActionController::Base` — no sessions,
  cookies, flash, or CSRF protection
- Consistent error shape: `{ error: "message" }` for single errors,
  `{ errors: {...} }` for validation errors
- Token auth via built-in `HttpAuthentication::Token` — no gems needed for
  simple cases

## RESTful Actions

```ruby
# app/controllers/api/v1/products_controller.rb
class Api::V1::ProductsController < Api::BaseController
  def index
    products = policy_scope(Product).page(params[:page]).per(params[:per_page])
    render json: {
      data: products.map { |p| ProductSerializer.new(p) },
      meta: pagination_meta(products)
    }
  end

  def show
    product = Product.find(params[:id])
    authorize product
    render json: { data: ProductSerializer.new(product) }
  end

  def create
    product = Product.new(product_params)
    authorize product

    if product.save
      render json: { data: ProductSerializer.new(product) }, status: :created
    else
      render json: { errors: product.errors }, status: :unprocessable_entity
    end
  end

  def update
    product = Product.find(params[:id])
    authorize product

    if product.update(product_params)
      render json: { data: ProductSerializer.new(product) }
    else
      render json: { errors: product.errors }, status: :unprocessable_entity
    end
  end

  def destroy
    product = Product.find(params[:id])
    authorize product

    product.destroy!
    head :no_content
  end

  private

  def product_params
    params.expect(product: [:name, :price, :description])
  end

  def pagination_meta(collection)
    {
      total: collection.total_count,
      page: collection.current_page,
      per_page: collection.limit_value,
      total_pages: collection.total_pages
    }
  end
end
```

**Conventions:**
- Wrap responses in `{ data: ... }` for consistency and extensibility
- Include `meta:` for pagination, not in the data array
- Use `head :no_content` for destroy (204)
- Authorize with Pundit — same policies as HTML controllers
- Serialize with `ProductSerializer` — see `shared/serializers.md`

## API Versioning

### URL Versioning (Preferred)

Simplest, most explicit, easiest to reason about:

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :products, only: [:index, :show, :create, :update, :destroy]
    resources :users, only: [:show]
  end

  namespace :v2 do
    resources :products, only: [:index, :show, :create, :update, :destroy]
  end
end
```

When adding v2, **don't copy-paste v1 controllers**. Instead, inherit and
override only what changed:

```ruby
class Api::V2::ProductsController < Api::V1::ProductsController
  # Only override actions with breaking changes
  def index
    products = policy_scope(Product).page(params[:page]).per(params[:per_page])
    render json: {
      data: products.map { |p| V2::ProductSerializer.new(p) },
      meta: pagination_meta(products)
    }
  end
end
```

### Header Versioning (Alternative)

Useful when URLs must stay stable (e.g., mobile app deep links):

```ruby
class Api::BaseController < ActionController::API
  before_action :set_api_version

  private

  def set_api_version
    @api_version = request.headers["API-Version"] || "v1"
  end
end
```

## Authentication

### Token Auth (Simple)

Built-in Rails token authentication — good for internal APIs and simple cases:

```ruby
# Base controller (shown above) uses:
include ActionController::HttpAuthentication::Token::ControllerMethods

def authenticate
  authenticate_or_request_with_http_token do |token, _options|
    @current_user = User.find_by(api_token: token)
  end
end
```

Generate tokens on the User model:

```ruby
class User < ApplicationRecord
  has_secure_token :api_token
end
```

### JWT (Stateless)

For stateless authentication across services. Use `Rails.application.credentials`
for the secret:

```ruby
# app/lib/json_web_token.rb
class JsonWebToken
  SECRET = Rails.application.credentials.secret_key_base

  def self.encode(payload, exp: 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET)
  end

  def self.decode(token)
    decoded = JWT.decode(token, SECRET).first
    HashWithIndifferentAccess.new(decoded)
  rescue JWT::DecodeError, JWT::ExpiredSignature
    nil
  end
end
```

```ruby
# app/controllers/api/auth_controller.rb
class Api::AuthController < Api::BaseController
  skip_before_action :authenticate, only: [:login]

  def login
    user = User.find_by(email: params[:email])

    if user&.authenticate(params[:password])
      token = JsonWebToken.encode(user_id: user.id)
      render json: { token: token }
    else
      render json: { error: "Invalid credentials" }, status: :unauthorized
    end
  end
end

# In Api::BaseController, swap token auth for JWT:
def authenticate
  header = request.headers["Authorization"]
  token = header&.split(" ")&.last
  decoded = JsonWebToken.decode(token)

  @current_user = User.find(decoded[:user_id]) if decoded
rescue ActiveRecord::RecordNotFound
  render json: { error: "Unauthorized" }, status: :unauthorized
end
```

**When to use which:**
- **Token auth** — internal APIs, admin tools, simple mobile backends
- **JWT** — multi-service architectures, stateless requirements, short-lived tokens
- **OAuth** — third-party integrations, user-facing API platforms (use Doorkeeper gem)

## Error Response Shape

Standardize error responses across the API:

```ruby
# Single error (auth failure, not found, bad request)
{ "error": "Record not found" }

# Validation errors (model errors)
{ "errors": { "name": ["can't be blank"], "price": ["must be greater than 0"] } }

# With error code (for client-side handling)
{ "error": "rate_limited", "message": "Too many requests", "retry_after": 60 }
```

## Rate Limiting

```ruby
# Gemfile
gem "rack-attack"

# config/initializers/rack_attack.rb
Rack::Attack.throttle("api/ip", limit: 60, period: 1.minute) do |req|
  req.ip if req.path.start_with?("/api/")
end

Rack::Attack.throttle("api/token", limit: 120, period: 1.minute) do |req|
  req.env["HTTP_AUTHORIZATION"]&.split(" ")&.last if req.path.start_with?("/api/")
end
```

## Testing

```ruby
class Api::V1::ProductsControllerTest < ActionDispatch::IntegrationTest
  def setup
    @user = users(:david)
    @token = @user.api_token
  end

  test "index returns paginated products" do
    get api_v1_products_path,
        headers: { "Authorization" => "Token token=#{@token}" }

    assert_response :ok
    json = response.parsed_body
    assert json.key?("data")
    assert json.key?("meta")
  end

  test "create returns 201 with valid params" do
    assert_difference "Product.count", 1 do
      post api_v1_products_path,
           params: { product: { name: "New", price: 9.99 } },
           headers: { "Authorization" => "Token token=#{@token}" },
           as: :json
    end

    assert_response :created
  end

  test "create returns 422 with invalid params" do
    post api_v1_products_path,
         params: { product: { name: "" } },
         headers: { "Authorization" => "Token token=#{@token}" },
         as: :json

    assert_response :unprocessable_entity
    assert response.parsed_body.key?("errors")
  end

  test "unauthenticated request returns 401" do
    get api_v1_products_path

    assert_response :unauthorized
  end
end
```

## See Also

- `shared/serializers.md` — serialization patterns (SimpleDelegator + AMS alternative)
- `shared/authorization.md` — Pundit policies (same policies for HTML and API)
- `shared/graphql.md` — GraphQL reference (not preferred, for reviewing existing APIs)
- `review-performance.md` — N+1 queries, HTTP caching, pagination
- `coding-classic.md § REST Mapping` — RESTful route design principles
