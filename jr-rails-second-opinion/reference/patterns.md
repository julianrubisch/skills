# Preferred Design Patterns

Your opinionated design toolkit. Each entry answers: what is it, when to reach for it,
and how you like it structured. Review agents use this as the "fix" vocabulary —
anti-patterns and smells point here for solutions.

## Query Objects

Encapsulate complex or reusable AR queries. Reach for this when a scope isn't enough
— multiple conditions, joins, subqueries, or shared across contexts.

### ApplicationQuery Base Class

Optional base class with convention-based model resolution. Namespace under
the model (`Model::SomeQuery`) and the base relation is inferred automatically:

```ruby
# app/queries/application_query.rb
class ApplicationQuery
  class << self
    def resolve(...) = new.resolve(...)

    def query_model
      name.sub(/::[^\:]+$/, "").safe_constantize
    end
  end

  private attr_reader :relation

  def initialize(relation = self.class.query_model&.all)
    @relation = relation
  end

  def resolve(...) = relation
end
```

### Relation-Wrapping Query

Wraps an AR relation. Composable — accepts a base scope, returns a scope.

```ruby
# app/queries/overdue_invoices_query.rb
class OverdueInvoicesQuery < ApplicationQuery
  def initialize(relation = Invoice.all)
    @relation = relation
  end

  def resolve
    relation
      .where("due_date < ?", Date.today)
      .where(paid: false)
      .order(:due_date)
  end
end

# Usage
OverdueInvoicesQuery.new.resolve
OverdueInvoicesQuery.new(current_user.invoices).resolve
```

### Parameterized Query

```ruby
class User::WithBookmarkedPostsQuery < ApplicationQuery
  def resolve(period: :previous_week)
    bookmarked_posts = build_bookmarked_posts_scope(period)
    relation.with(bookmarked_posts:).joins(:bookmarked_posts)
  end

  private

  def build_bookmarked_posts_scope(period)
    Post.public_send(period)
        .where.associated(:bookmarks)
        .select(:user_id).distinct
  end
end

# Default relation (User.all) inferred from namespace
User::WithBookmarkedPostsQuery.resolve
User::WithBookmarkedPostsQuery.new(account.users).resolve(period: :this_month)
```

### Arel for Composable Fragments

Use Arel for reusable query fragments — avoids SQL string stitching:

```ruby
class Post::SearchQuery < ApplicationQuery
  def resolve(query:)
    relation.where(
      Post.arel_table[:title].matches("%#{query}%")
        .or(Post.arel_table[:body].matches("%#{query}%"))
    )
  end
end
```

### Attaching Query Objects as Scopes

```ruby
class ApplicationRecord < ActiveRecord::Base
  def self.query(query_class, ...)
    query_class.new(all).resolve(...)
  end
end

# Usage — composable with other scopes
Post.published.query(Post::TrendingQuery, min_views: 50)
```

### Search / Filter Form

**Preferred** when query parameters come from user input. Uses `ActiveModel::Model`
\+ `ActiveModel::Attributes` for type coercion, defaults, and `form_with` integration.
Lives in `app/models/`.

```ruby
# app/models/post_search.rb
class PostSearch
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :query, :string
  attribute :author_id, :integer
  attribute :status, :string
  attribute :sort_column, :string, default: "created_at"
  attribute :sort_direction, :string, default: "desc"

  def results
    scope = Post.all
    scope = scope.where("title ILIKE ?", "%#{query}%") if query.present?
    scope = scope.where(author_id: author_id) if author_id.present?
    scope = scope.where(status: status) if status.present?
    scope.order(sort_column => sort_direction)
  end

  def sort_options
    [
      ["Newest First", "created_at-desc"],
      ["Oldest First", "created_at-asc"],
      ["Title A-Z", "title-asc"],
      ["Title Z-A", "title-desc"]
    ]
  end
end
```

### Atomic vs Complex Scopes

**Atomic scopes stay in the model** — single condition, composable:

```ruby
class Post < ApplicationRecord
  scope :published, -> { where.not(published_at: nil) }
  scope :recent, -> { where(created_at: 1.week.ago..) }
  scope :by_author, ->(user) { where(user:) }
end

# Compose them
Post.published.recent.by_author(user)
```

**Complex scopes extract to query objects** — multiple conditions, context-specific:

```ruby
# BAD: Complex scope in model
class Post < ApplicationRecord
  scope :trending, -> {
    joins(:views, :comments)
      .where(views: {created_at: 1.day.ago..})
      .group(:id)
      .having("COUNT(views.id) > 100")
      .order("COUNT(comments.id) DESC")
  }
end

# GOOD: Query object
class Post::TrendingQuery < ApplicationQuery
  def resolve(min_views: 100, period: 1.day.ago..)
    relation
      .joins(:views, :comments)
      .where(views: {created_at: period})
      .group(:id)
      .having("COUNT(views.id) > ?", min_views)
      .order("COUNT(comments.id) DESC")
  end
end
```

### Anti-patterns

**Over-scoping — conflicting orders:**

```ruby
# BAD: Scopes with implicit ordering
scope :recent, -> { order(created_at: :desc) }
scope :popular, -> { order(views_count: :desc) }

Post.recent.popular  # Which order wins?
```

**Context-free queries in models:**

```ruby
# BAD: This is really "admin dashboard search"
class User < ApplicationRecord
  scope :search, ->(q) {
    joins(:posts, :comments)
      .where("users.name LIKE ?", "%#{q}%")
      .or(where("posts.title LIKE ?", "%#{q}%"))
  }
end

# GOOD: Context-specific query object
class Admin::UserSearchQuery < ApplicationQuery
  # ...
end
```

### Related Gems

| Gem | Purpose |
|-----|---------|
| [arel-helpers](https://github.com/camertron/arel-helpers) | Reduce Arel boilerplate for complex JOINs |

**When:** inline `.where` chains repeated in 2+ places, or query has 3+ conditions.
**When not:** a simple named scope covers it.

## Repositories

**Edge case pattern** — repositories group cohesive sets of complex queries behind
a clean interface. They complement Active Record rather than replacing it. This goes
against vanilla Rails conventions, so reach for it only when the complexity justifies
the abstraction.

**The heuristic:** use a repository when you have 3+ complex queries that clearly
belong to a single topic or concern (a dashboard, an analytics view, an activity
feed). If the queries don't form a cohesive group, use individual query objects
or scopes instead.

### When to Use

- Multiple related complex queries belonging to a clear concern (dashboard, analytics, activity feed)
- Cross-model aggregation and reporting
- External data source integration alongside local queries
- Caching strategies for expensive read operations

### When NOT to Use

- Simple single-model queries — use scopes
- CRUD operations — use Active Record directly
- Individual complex queries — use query objects
- Before the complexity actually materializes — don't pre-abstract

### Dashboard Repository

```ruby
class DashboardRepository
  def initialize(user)
    @user = user
  end

  def stats
    Rails.cache.fetch(cache_key, expires_in: 5.minutes) do
      {
        total_posts: user_posts.count,
        published_posts: user_posts.published.count,
        total_views: total_views,
        engagement_rate: calculate_engagement
      }
    end
  end

  private

  attr_reader :user

  def cache_key
    "dashboard/#{user.id}/#{user.updated_at.to_i}"
  end

  def user_posts
    @user_posts ||= Post.where(author: user)
  end

  def total_views
    user_posts.sum(:views_count)
  end

  def calculate_engagement
    return 0 if total_views.zero?
    (Comment.where(post: user_posts).count.to_f / total_views * 100).round(2)
  end
end
```

### Activity Feed Repository

Aggregates data from multiple models into a unified feed:

```ruby
class UserActivityRepository
  def initialize(user)
    @user = user
  end

  def recent_activity
    activities = []
    activities.concat(recent_posts)
    activities.concat(recent_comments)
    activities.concat(recent_likes)
    activities.sort_by(&:created_at).reverse.first(20)
  end

  private

  attr_reader :user

  def recent_posts
    user.posts.recent.limit(10).map do |post|
      Activity.new(type: :post, record: post,
        created_at: post.created_at, description: "Published '#{post.title}'")
    end
  end

  def recent_comments
    user.comments.recent.limit(10).map do |comment|
      Activity.new(type: :comment, record: comment,
        created_at: comment.created_at, description: "Commented on '#{comment.post.title}'")
    end
  end

  def recent_likes
    user.likes.recent.includes(:post).limit(10).map do |like|
      Activity.new(type: :like, record: like,
        created_at: like.created_at, description: "Liked '#{like.post.title}'")
    end
  end

  Activity = Data.define(:type, :record, :created_at, :description)
end
```

### Read Model Repository

For complex reporting:

```ruby
class AnalyticsRepository
  def post_performance(date_range:)
    Post
      .where(published_at: date_range)
      .joins("LEFT JOIN comments ON comments.post_id = posts.id")
      .joins("LEFT JOIN likes ON likes.post_id = posts.id")
      .group("DATE(posts.published_at)")
      .select(
        "DATE(posts.published_at) AS date",
        "COUNT(DISTINCT posts.id) AS posts_count",
        "COUNT(DISTINCT comments.id) AS comments_count",
        "COUNT(DISTINCT likes.id) AS likes_count",
        "SUM(posts.views_count) AS total_views"
      )
      .order(:date)
      .map { |row| PerformanceData.new(row.attributes.symbolize_keys) }
  end

  PerformanceData = Data.define(:date, :posts_count, :comments_count, :likes_count, :total_views) do
    def engagement_rate
      return 0 if total_views.zero?
      ((comments_count + likes_count).to_f / total_views * 100).round(2)
    end
  end
end
```

### External Data Integration

```ruby
class BookRepository
  def initialize(api_client: GoogleBooksAPI.new)
    @api_client = api_client
  end

  def find_with_external_data(isbn)
    book = Book.find_by!(isbn: isbn)
    external_data = fetch_external_data(isbn)

    BookWithMetadata.new(
      book: book, cover_url: external_data[:cover],
      description: external_data[:description], rating: external_data[:average_rating]
    )
  end

  private

  attr_reader :api_client

  def fetch_external_data(isbn)
    Rails.cache.fetch("book_external/#{isbn}", expires_in: 1.day) do
      api_client.fetch(isbn)
    end
  end

  BookWithMetadata = Data.define(:book, :cover_url, :description, :rating) do
    delegate_missing_to :book
  end
end
```

### Repository vs Query Object

| Repository | Query Object |
|------------|--------------|
| Multiple related queries | Single query concern |
| Cross-model aggregation | Single-model filtering |
| External data integration | Database queries only |
| Caching strategies | Composable query logic |
| Read model concerns | Reusable query fragments |

### Testing

```ruby
class DashboardRepositoryTest < ActiveSupport::TestCase
  def setup
    @user = users(:author)
    @repository = DashboardRepository.new(@user)
  end

  test "#stats returns correct post counts" do
    stats = @repository.stats

    assert_equal @user.posts.count, stats[:total_posts]
    assert_equal @user.posts.published.count, stats[:published_posts]
  end

  test "#stats caches results" do
    first_result = @repository.stats
    second_result = @repository.stats

    assert_equal first_result, second_result
  end
end
```

### Anti-patterns

**Thin wrapper over Active Record:**

```ruby
# BAD: No value added
class PostRepository
  def find(id) = Post.find(id)
  def all = Post.all
  def create(params) = Post.create(params)
end

# GOOD: Just use Active Record directly
Post.find(id)
```

**Business logic in repository:**

```ruby
# BAD: Repository doing mutations and side effects
class PostRepository
  def publish(post)
    post.update!(published_at: Time.current)
    NotificationService.notify_subscribers(post)
  end
end

# GOOD: Repository for reads, model methods for mutations
class PostRepository
  def published_with_stats
    # Complex read query
  end
end
```

### File Organization

```
app/
├── models/
│   └── post.rb
├── queries/
│   └── post/
│       └── trending_query.rb
└── repositories/
    ├── dashboard_repository.rb
    ├── analytics_repository.rb
    └── user_activity_repository.rb
```

**When:** 3+ complex queries form a cohesive group around a single concern (dashboard,
analytics, activity feed), especially with caching or external data.
**When not:** individual complex queries (use query objects), simple queries (use scopes),
or when the grouping feels forced. Don't pre-abstract — let the need emerge.

## Filter Objects

Transform datasets based on user-provided request parameters. Presentation layer —
consumes controller params, applies transformations to domain collections. Can use
query objects internally (filters orchestrate, queries implement).

### Filter vs Query Object

| Aspect | Filter Object | Query Object |
|--------|---------------|--------------|
| Layer | Presentation | Domain |
| Input | Request params | Domain values |
| Purpose | UI filtering/sorting | Reusable queries |
| Location | `app/filters/` | `app/queries/` or `app/models/` |

**Rule of thumb:** if the logic depends on `params`, it's a filter. If it encapsulates
a reusable business query, it's a query object.

### Standalone Filter Object

```ruby
# app/filters/projects_filter.rb
class ProjectsFilter
  ALLOWED_SORT_FIELDS = %w[name created_at].freeze

  def initialize(relation, params)
    @relation = relation
    @params = params
  end

  def filter
    result = @relation
    result = filter_by_status(result)
    result = filter_by_search(result)
    result = apply_sorting(result)
    result
  end

  private

  def filter_by_status(relation)
    return relation unless @params[:status].present?
    relation.where(status: @params[:status])
  end

  def filter_by_search(relation)
    return relation unless @params[:q].present?
    relation.where("name ILIKE ?", "%#{@params[:q]}%")
  end

  def apply_sorting(relation)
    field = @params[:sort_by]
    return relation unless ALLOWED_SORT_FIELDS.include?(field)
    relation.order(field => @params[:sort_order] || :asc)
  end
end
```

Controller usage:

```ruby
class ProjectsController < ApplicationController
  def index
    @projects = ProjectsFilter.new(Project.all, filter_params).filter
  end

  private

  def filter_params
    params.slice(:status, :q, :sort_by, :sort_order)
          .permit(:status, :q, :sort_by, :sort_order)
  end
end
```

### Security

Always allowlist at both layers:

```ruby
# Controller — only permit known filter keys
def filter_params
  params.slice(:status, :q).permit(:status, :q)
end

# Filter object — allowlist enum values
ALLOWED_STATUSES = %w[draft published].freeze

def filter_by_status(relation)
  return relation unless ALLOWED_STATUSES.include?(@params[:status])
  relation.where(status: @params[:status])
end

# Sorting — allowlist column names to prevent SQL injection
ALLOWED_SORT_FIELDS = %w[name created_at].freeze
```

### Gem alternative: has_scope

[has_scope](https://github.com/heartcombo/has_scope) (Heartcombo / Devise org)
provides declarative param-to-scope mapping in controllers. Recommended when
filter objects feel like overkill but inline `params[:x]` chains are growing:

```ruby
class ProjectsController < ApplicationController
  has_scope :status
  has_scope :search, as: :q
  has_scope :sort_by, using: [:column, :direction], type: :hash

  def index
    @projects = apply_scopes(Project).all
  end
end
```

### Anti-patterns

- **Filtering in controller** — inline `.where` chains based on params; extract to a filter object or use `has_scope`
- **Universal filter object** — one god filter for all interfaces; create interface-specific filters instead (`Admin::UsersFilter`, `Api::V1::UsersFilter`)

**When:** 3+ optional filter params in a controller action, or sorting/pagination logic mixed with filtering.
**When not:** 1-2 simple scopes — inline or `has_scope` is enough.

## Value Objects

Represent a domain concept with no identity — **fungible** objects whose equality
is based entirely on their attributes, not an id. Two `Color.new(255, 0, 0)`
instances are the same color. Small, immutable, and replaceable.
Reach for this when you see data clumps or primitive obsession.

**When to use:**
- Domain concepts like Money, Address, Coordinates, MediaType
- Groups of related attributes that travel together
- Attributes with behavior (formatting, validation, comparison)
- JSON/JSONB column wrappers with validation

**When NOT to use:**
- Entities with identity (needs its own table → use a model)
- Simple scalar values without behavior

### Using `Data.define` (Preferred for Simple VOs)

Ruby's `Data.define` gives you immutability, equality, and pattern matching for
free — the simplest way to build value objects:

```ruby
class MediaType < Data.define(:content_type)
  SVG_TYPES = %w[image/svg image/svg+xml].freeze
  FONT_TYPES = %w[font/otf font/ttf font/woff font/woff2].freeze

  include Comparable

  def <=>(other)
    content_type <=> other.content_type
  end

  def video? = content_type.start_with?("video")
  def image? = content_type.start_with?("image")
  def svg?   = SVG_TYPES.include?(content_type)
  def font?  = FONT_TYPES.include?(content_type)
end

# Usage
media_type = MediaType.new("image/png")
media_type.image?  #=> true
media_type.video?  #=> false
```

```ruby
class Money < Data.define(:amount_cents, :currency)
  CURRENCIES = %w[USD EUR GBP].freeze

  def initialize(amount_cents:, currency: "USD")
    raise ArgumentError unless CURRENCIES.include?(currency)
    super
  end

  def to_s = format("%.2f %s", amount_cents / 100.0, currency)

  def +(other)
    raise ArgumentError unless currency == other.currency
    Money.new(amount_cents: amount_cents + other.amount_cents, currency:)
  end

  def *(multiplier)
    Money.new(amount_cents: (amount_cents * multiplier).round, currency:)
  end
end

# Usage
price = Money.new(amount_cents: 1999, currency: "USD")
price.to_s  #=> "19.99 USD"
price * 2   #=> Money(amount_cents: 3998, currency: "USD")
```

### With ActiveModel (When Validation Needed)

When you need `validates`, `ActiveModel::Attributes`, or form integration, reach
for ActiveModel. Trade-off: ~4-13x slower than plain Data, but only optimize when
profiling shows it's a bottleneck.

```ruby
# app/models/color.rb — value object with equality, conversions, comparability
class Color
  include Comparable
  include ActiveModel::Validations

  attr_reader :red, :green, :blue, :alpha

  validates :red, :green, :blue, inclusion: { in: 0..255 }
  validates :alpha, inclusion: { in: 0.0..1.0 }

  def initialize(red_or_hex, green = nil, blue = nil, alpha = 1.0)
    case [red_or_hex, green, blue]
    in [/\A#?[0-9A-Fa-f]{6}\z/ => hex, nil, nil]
      @red, @green, @blue = [hex.delete("#")].pack("H*").unpack("C*")
    else
      @red, @green, @blue = red_or_hex.to_i, green.to_i, blue.to_i
    end
    @alpha = alpha.to_f
  end

  def ==(other)
    other.is_a?(Color) && hash == other.hash
  end

  def hash
    [red, green, blue, alpha].hash
  end

  def <=>(other)
    lightness <=> other.lightness
  end

  def to_hex = format("%02X%02X%02X", red, green, blue)
  def to_rgb_s = "rgb(#{red} #{green} #{blue})"
end
```

### Integrating with ActiveRecord via `composed_of`

`composed_of` is a built-in Rails macro that maps denormalized columns to a
value object — think of it as an inline `has_one` without a separate table.

```ruby
class Theme < ApplicationRecord
  composed_of :primary_color,
              class_name: "Color",
              mapping: {
                primary_color_red: :red,
                primary_color_green: :green,
                primary_color_blue: :blue,
                primary_color_alpha: :alpha
              },
              converter: ->(value) { Color.new(value) }

  validates_associated :primary_color
end

# Now you get clean assignment, querying, and form integration:
theme = Theme.create!(name: "Dark", primary_color: "#FF0000")
theme.primary_color.to_hex          # => "FF0000"
theme.update!(primary_color: "#00FF00")
Theme.where(primary_color: Color.new("#0000FF"))
```

**Key options:**
- `mapping:` — column-to-attribute hash. **Order matters** — it determines
  constructor argument order.
- `converter:` — proc called on assignment so you can write `= "#FF0000"`
  instead of `= Color.new("#FF0000")`.
- `allow_nil:` — permits setting the value object to nil (all columns → NULL).

### With JSON Store

For JSON/JSONB columns, wrap the store in a value object with validations:

```ruby
class User < ApplicationRecord
  store :address, coder: JSON

  def address
    @address ||= User::Address.new(super || {})
  end

  validate do |record|
    next if address.valid?
    record.errors.add(:address, "is invalid")
  end
end

class User::Address
  include ActiveModel::API
  include ActiveModel::Attributes

  attribute :country
  attribute :city
  attribute :street
  attribute :zip

  validates :country, :zip, presence: true

  def full_address
    [street, city, zip, country].compact.join(", ")
  end
end

# Usage
user = User.create!(address: {country: "USA", city: "Bronx", zip: "10463"})
user.address.country       #=> "USA"
user.address.full_address  #=> "Bronx, 10463, USA"
```

### Anti-Patterns

**Mutable value objects:**

```ruby
# BAD
class Address
  attr_accessor :city, :street  # Mutable!
end

# GOOD: Use Data class (immutable)
class Address < Data.define(:city, :street)
end
```

**Value objects with side effects:**

```ruby
# BAD
class Coordinates < Data.define(:lat, :lng)
  def save_to_cache!
    Rails.cache.write("coords", self)
  end
end

# GOOD: Value objects are pure
class Coordinates < Data.define(:lat, :lng)
  def distance_to(other)
    # Pure calculation
  end
end
```

**When:** same 2-3 primitive values always travel together, or you find yourself
duplicating formatting/comparison logic for raw primitives. Prefer `Data.define`
for simple cases; reach for ActiveModel when you need validations or form
integration.
**When not:** the concept has its own identity (needs its own table → use a model),
or it's a simple scalar without behavior.

## Form Objects

Handle multi-model forms, virtual attributes, or complex input validation outside
the model. Keeps models clean of presentation-driven concerns. A form object
models user interaction, not domain entities — it straddles the presentation /
application boundary (see `shared/architecture.md`).

Typical signals that a form object is needed:
- `before|after_create|update` hooks with side effects on the model
- Conditional validation based on UI flow state
- Transient attributes (`should_send_welcome_email`) without database backing
- A model reaching out to mutate other models from within its own logic

### ApplicationForm Base Class

Extract common form plumbing into a base class:

```ruby
# app/forms/application_form.rb
class ApplicationForm
  include ActiveModel::Model
  include ActiveModel::Attributes
  extend ActiveModel::Callbacks

  define_model_callbacks :save, only: :after
  define_model_callbacks :commit, only: :after

  class << self
    def after_save(...)
      set_callback(:save, :after, ...)
    end

    def after_commit(...)
      set_callback(:commit, :after, ...)
    end

    # Quack like ActiveRecord for route helpers and form_with
    def model_name
      ActiveModel::Name.new(self, nil, name.sub(/Form$/, ""))
    end
  end

  # Behaves like ActiveRecord: returns false on invalid, wraps in transaction
  def save
    return false unless valid?

    with_transaction do
      AfterCommitEverywhere.after_commit { run_callbacks(:commit) }
      run_callbacks(:save) { submit! }
    end
  end

  private

  def with_transaction(&)
    ApplicationRecord.transaction(&)
  end

  # Subclasses must implement — the actual persistence logic
  def submit!
    raise NotImplementedError
  end
end
```

**Callback distinction:**
- `after_save` — runs inside the transaction (create related records, update counters)
- `after_commit` — runs after transaction commits (send emails, enqueue jobs).
  Requires [`after_commit_everywhere`](https://github.com/Envek/after_commit_everywhere) gem.

### Factory Method

Each form defines a `.for` class method that permits its own params — keeps
strong parameter logic co-located with the form:

```ruby
class ContactForm < ApplicationForm
  class << self
    def for(params)
      new(params.permit(:name, :email, :should_send_welcome_email, :follow_up))
    end
  end
end

# Controller usage:
@form = ContactForm.for(params[:contact])
```

### Concrete Form Object

```ruby
# app/forms/contact_form.rb
class ContactForm < ApplicationForm
  attribute :name, :string
  attribute :email, :string
  attribute :should_send_welcome_email, :boolean, default: false
  attribute :follow_up, :boolean, default: false

  validates :name, presence: true, if: :follow_up
  validates :email, presence: true
  validate :contact_is_valid

  after_commit :deliver_welcome_email!, if: :should_send_welcome_email

  class << self
    def for(params)
      new(params.permit(:name, :email, :should_send_welcome_email, :follow_up))
    end
  end

  delegate :to_param, :id, to: :contact, allow_nil: true

  def contact
    @contact ||= Contact.new(name:, email:,
      follow_up_started_at: (follow_up ? Time.current : nil))
  end

  private

  def submit!
    contact.save!
  end

  # Bubble model-level errors into the form
  def contact_is_valid
    return if contact.valid?
    errors.merge!(contact.errors)
  end

  def deliver_welcome_email!
    ContactMailer.welcome(name, email).deliver_later
  end
end
```

### Multi-Model Form

```ruby
class RegistrationForm < ApplicationForm
  attribute :name, :string
  attribute :email, :string
  attribute :project_name, :string
  attribute :should_create_project, :boolean

  validates :project_name, presence: true, if: :should_create_project
  validate :user_is_valid

  attr_reader :user

  after_save :create_initial_project, if: :should_create_project

  def initialize(...)
    super
    @user = User.new(email:, name:)
  end

  private

  def submit!
    user.save!
  end

  def create_initial_project
    user.projects.create!(name: project_name)
  end

  def user_is_valid
    return if user.valid?
    user.errors.each do |error|
      errors.add(error.attribute, error.message)
    end
  end
end
```

### Model-less Form

```ruby
class FeedbackForm < ApplicationForm
  attribute :message, :string
  attribute :email, :string
  attribute :category, :string

  validates :message, presence: true, length: { minimum: 10 }
  validates :email, presence: true

  after_commit :deliver_feedback

  private

  def submit!
    true  # No model to save
  end

  def deliver_feedback
    FeedbackMailer.new_feedback(email:, message:, category:).deliver_later
  end
end
```

### Wizard Forms (Multi-Step)

Use a state machine for complex multi-step forms. Validate only the current step
(see `shared/state_machines.md` for full AASM reference):

```ruby
class OnboardingForm < ApplicationForm
  include AASM

  aasm column: :step do
    state :profile, initial: true
    state :preferences
    state :confirmation

    event :next_step do
      transitions from: :profile, to: :preferences
      transitions from: :preferences, to: :confirmation
    end

    event :prev_step do
      transitions from: :preferences, to: :profile
      transitions from: :confirmation, to: :preferences
    end
  end

  validates :name, presence: true, if: :profile?
  validates :email, presence: true, if: :profile?
  validates :theme, presence: true, if: :preferences?

  def submit!
    return true unless confirmation?
    User.create!(attributes.except(:step))
  end
end
```

### Usage in Controller and View

```ruby
# Controller — same shape as a model-backed controller
class ContactsController < ApplicationController
  def new
    @contact_form = ContactForm.new
  end

  def create
    @contact_form = ContactForm.for(params[:contact])
    if @contact_form.save
      redirect_to @contact_form   # routes to /contacts/:id via model_name
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

```erb
<%# form_with routes to /contacts via model_name %>
<%= form_with model: @contact_form do |f| %>
  <%= f.text_field :name %>
  <%= f.email_field :email %>
  <%= f.check_box :should_send_welcome_email %>
<% end %>
```

### Anti-patterns

**Duplicating model validations:**

```ruby
# BAD — duplicates User validation
class UserForm < ApplicationForm
  validates :email, presence: true, uniqueness: true
end

# GOOD — delegate to model, merge errors
class UserForm < ApplicationForm
  validate :user_is_valid

  def user_is_valid
    return if user.valid?
    user.errors.each { |e| errors.add(e.attribute, e.message) }
  end
end
```

**UI logic in model callbacks:**

```ruby
# BAD — model callback for UI-specific behavior
class User < ApplicationRecord
  after_create :send_welcome_email, if: :from_registration_form?
end

# GOOD — form handles UI-specific side effects
class RegistrationForm < ApplicationForm
  after_commit :send_welcome_email
end
```

### Related Gems

| Gem | Purpose |
|-----|---------|
| [after_commit_everywhere](https://github.com/Envek/after_commit_everywhere) | `after_commit` callbacks outside Active Record |

**When:** form spans multiple models, has virtual fields, validation rules are
specific to one UI flow, or the model carries transient attributes / side-effect
callbacks that belong to the interaction, not the domain.
**When not:** single-model form with standard validations — just use the model.

## Rule Objects

Encapsulate complex conditional logic — sets of guard clauses that determine
whether something should happen. Better than a long `call` method with many
`return if` checks. Testable in isolation.

```ruby
# app/rules/continuation_rule.rb
class ContinuationRule
  CONVERSATION_LENGTH_MIN = 3
  LENGTH_CUTOFF = 250
  MAX_TOKENS = 8_000

  def initialize(message)
    @message = message
  end

  def satisfied?
    !sender_is_csm? &&
      onboarding_complete? &&
      conversation_long_enough? &&
      message_short_enough? &&
      within_token_limit?
  end

  private

  def sender_is_csm?       = @message.sender.csm?
  def onboarding_complete? = @message.account.onboarding_complete?
  def conversation_long_enough? = @message.conversation.messages.count >= CONVERSATION_LENGTH_MIN
  def message_short_enough? = @message.content.to_s.length <= LENGTH_CUTOFF
  def within_token_limit?  = @message.tokens_count.to_i <= MAX_TOKENS
end

# Usage
if ContinuationRule.new(message).satisfied?
  # proceed
end
```

**When:** a single method has 4+ guard clauses, or the conditions need to be
tested independently, or the same set of conditions appears in multiple places.

## Strategy Objects

Composition-based polymorphism. Instead of subclassing to vary behavior,
inject collaborator objects that implement a common interface. The host object
delegates to its strategy — behavior is pluggable at runtime.

```ruby
# Each strategy implements the same interface
class Transport::Email
  def deliver(campaign)
    campaign.addressees.find_each do |addressee|
      EmailService::Client.deliver(to: addressee, body: campaign.body)
    end
  end
end

class Transport::Sms
  def deliver(campaign)
    campaign.addressees.find_each do |addressee|
      campaign.body.chars.each_slice(SMS_CHAR_LENGTH).map(&:join).each do |chunk|
        SmsService::Client.deliver(to: addressee, body: chunk)
      end
    end
  end
end

class Output::HTML
  def format(content) = content.to_html
end

class Output::Plain
  def format(content) = content.truncate(SMS_CHAR_LENGTH)
end

# Host composes strategies — no inheritance needed
class Campaign
  attr_accessor :transport, :output
  attr_reader :body

  def initialize(transport:, output:, body:)
    @transport, @output, @body = transport, output, body
  end

  def process
    @body = output.format(@body)
    transport.deliver(self)
  end
end

# Runtime flexibility — swap strategies freely
campaign = Campaign.new(transport: Transport::Email.new, output: Output::HTML.new, body: content)
campaign.process

campaign.transport = Transport::Sms.new
campaign.output    = Output::Plain.new
campaign.process
```

**Rule of thumb** (Sandi Metz):
- **Inherit** only for true "is-a" relationships with a stable type hierarchy.
- **Mixin** for cross-cutting "acts-as" concerns (`Closeable`, `Watchable`).
- **Compose** for flexible "uses-a" relationships — when you need runtime
  swappability or the behaviors vary independently.

If you're tempted to inherit just to reuse code, compose instead.

**When:** family of interchangeable behaviors (transports, formatters, storage
backends). The behaviors are independent and don't need deep access to the
host's internal state.
**When not:** simple `case` in a controller action that doesn't warrant the
abstraction. If the strategy needs half the model's attributes, you're adding
indirection for no gain (Feature Envy).

## `store_accessor` for JSON/JSONB Columns

Expose JSON column keys as first-class attributes with type coercion.
Avoids `properties["reactions"]` string-keyed access throughout the codebase.

```ruby
class Message < ApplicationRecord
  store_accessor :properties, :reactions, :responding_to, :tokens_count

  # Now accessed as:
  message.reactions        # instead of message.properties["reactions"]
  message.responding_to=   # instead of message.properties[:responding_to] =
end
```

**When:** a JSONB/JSON column has a known set of keys accessed in multiple places.
**When not:** the JSON structure is fully dynamic or schema-less.

## Presenters / Decorators

Add display logic to a model without polluting it. Keeps views and models clean.
Two flavors: **open** (decorator, delegates everything) and **closed** (explicit
interface, strict isolation). Choose based on how much of the model the view needs.

> For API JSON responses, use **serializers** instead — same `SimpleDelegator`
> pattern but targeting `#as_json`. See `shared/serializers.md`.
>
> When a presenter starts building HTML (`content_tag`, `tag.*`), extract to a
> **Phlex component** instead. See `shared/components.md § Extraction Signals`.

### Open Presenter (Decorator)

Uses `SimpleDelegator` — the presenter *is* the model (all methods delegate through)
but layers on view-specific formatting. The `h` accessor gives access to view helpers
(routes, `link_to`, `number_to_currency`, etc.) when needed.

```ruby
# app/presenters/base_presenter.rb
class BasePresenter < SimpleDelegator
  def initialize(model, view = nil)
    @view = view
    super(model)
  end

  def h
    @view
  end
end
```

```ruby
# app/presenters/contact_presenter.rb
class ContactPresenter < BasePresenter
  def recipient
    recipient = "z.Hd. #{title.title} #{titleinfo} #{firstname} #{lastname}"
    recipient << ", #{postnomen}" if postnomen.present?
    recipient
  end

  def recipient_salutation
    salutation = "Sehr geehrte#{"r" if title.title == "Herr"}"

    "#{salutation} #{title.title} #{titleinfo} #{firstname} #{lastname}"
  end
end
```

```ruby
# app/presenters/searchprofile_presenter.rb
class SearchprofilePresenter < BasePresenter
  include ContactsHelper

  def delivery_string
    case fk_delivery
    when 1
      "Fax: #{recipient&.faxnumber&.present? ? format_tel_number(recipient, 'fax') : ''}"
    when 2
      "Per Post"
    when 3
      recipient.email.present? ? recipient.email : "E-Mail"
    end
  end

  def area_description
    %i[officearea parcelarea parkspace storagearea totalarea warehausearea].map do |area_type|
      next unless present_area_types.keys.any? { |key| key.match? /#{area_type}/ }

      from = send("#{area_type}from")
      to = send("#{area_type}to")
      from_string = " von #{from} qm" if from.present?
      to_string = " bis #{to} qm" if to.present?

      "#{Searchprofile.human_attribute_name(area_type)}:#{from_string}#{to_string}"
    end.compact.join(", ")
  end
end
```

### Closed Presenter (Explicit Interface)

Best for strict isolation — only expose what the view actually needs:

```ruby
# app/presenters/user_presenter.rb
class UserPresenter
  delegate :id, :to_model, to: :user

  private attr_reader :user

  def initialize(user)
    @user = user
  end

  def short_name
    user.name.squish.split(/\s/).then do |parts|
      parts[0..-2].map { _1[0] + "." }.join + parts.last
    end
  end

  def status_badge_class
    case user.status
    when "active" then "badge-success"
    when "pending" then "badge-warning"
    else "badge-secondary"
    end
  end

  def member_since
    user.created_at.strftime("%B %Y")
  end
end
```

### Multi-Model Presenter

For composite UI elements that combine data from multiple sources:

```ruby
class User::BookPresenter < SimpleDelegator
  private attr_reader :book_read

  delegate :read?, :read_at, :score, to: :book_read

  def initialize(book, book_read)
    super(book)
    @book_read = book_read
  end

  def progress_icon
    read? ? "fa-circle-check" : "fa-clock"
  end

  def score_class
    case score
    when 0..2 then "text-red-600"
    when 3...4 then "text-yellow-600"
    when 4.. then "text-green-600"
    end
  end
end
```

### Helper

Available in both controllers and views via a concern:

```ruby
# app/controllers/concerns/presentable.rb
module Presentable
  extend ActiveSupport::Concern

  included do
    helper_method :present
  end

  def present(model)
    klass = "#{model.class}Presenter".constantize
    presenter = klass.new(model, view_context)
    block_given? ? yield(presenter) : presenter
  end
end

# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Presentable
end
```

### Usage in Controller and View

```ruby
# Controller — wrap early, pass presenter as the ivar
class ContactsController < ApplicationController
  def show
    @contact = present(Contact.find(params[:id]))
  end
end
```

```erb
<%# View — presenter methods alongside regular model attributes %>
<h1><%= @contact.recipient %></h1>
<p><%= @contact.recipient_salutation %></p>
<p><%= @contact.email %></p>  <%# delegates to Contact#email %>

<%# Or use the block form inline via helper_method %>
<% present(@searchprofile) do |sp| %>
  <p><%= sp.delivery_string %></p>
  <p><%= sp.area_description %></p>
<% end %>

<%# Closed presenter — same helper works %>
<% present(@user) do |p| %>
  <span class="<%= p.status_badge_class %>">
    <%= p.short_name %>
  </span>
  <small>Member since <%= p.member_since %></small>
<% end %>
```

### Anti-patterns

**Representation logic in models:**

```ruby
# BAD
class User < ApplicationRecord
  def status_badge_class
    case status
    when "active" then "badge-success"
    # ...
    end
  end
end

# GOOD: Move to presenter
class UserPresenter
  def status_badge_class
    # ...
  end
end
```

**Leaking decorators to lower layers:**

```ruby
# BAD: Presenter passed to service
def create
  @user = UserPresenter.new(User.find(params[:id]))
  SomeService.call(@user)  # Presenter leaked to application layer!
end

# GOOD: Present only in views
def show
  @user = User.find(params[:id])
end
```

**Global helpers with prefixed methods:**

```ruby
# BAD: Polluting helper namespace
module UsersHelper
  def user_short_name(user) = # ...
  def user_status_badge(user) = # ...
end

# GOOD: Encapsulate in presenter
class UserPresenter
  def short_name = # ...
  def status_badge_class = # ...
end
```

### Testing

```ruby
class UserPresenterTest < ActiveSupport::TestCase
  def setup
    @user = User.new(name: "John Michael Doe", status: "active",
                     created_at: Time.zone.local(2024, 1, 15))
    @presenter = UserPresenter.new(@user)
  end

  test "#short_name abbreviates middle names" do
    assert_equal "J.M.Doe", @presenter.short_name
  end

  test "#status_badge_class returns success for active users" do
    assert_equal "badge-success", @presenter.status_badge_class
  end

  test "#status_badge_class returns warning for pending users" do
    @user = User.new(name: "Jane Doe", status: "pending")
    presenter = UserPresenter.new(@user)
    assert_equal "badge-warning", presenter.status_badge_class
  end

  test "#member_since formats creation date" do
    assert_equal "January 2024", @presenter.member_since
  end
end
```

**When:** model methods start returning HTML, formatted strings, or view-specific logic.
**When not:** the formatting is a one-liner used in a single view — inline is fine.

## Policy Objects (Pundit)

Authorization rules live in policy objects, separate from models and controllers.
One policy per resource, named `<Model>Policy`. Policies belong to the application
layer — between presentation (enforcement) and domain (entities).

### ApplicationPolicy Base Class

```ruby
# app/policies/application_policy.rb
class ApplicationPolicy
  attr_reader :user, :record

  def initialize(user, record)
    @user = user
    @record = record
  end

  def index?   = false
  def show?    = false
  def create?  = false
  def new?     = create?
  def update?  = false
  def edit?    = update?
  def destroy? = false

  class Scope
    def initialize(user, scope)
      @user = user
      @scope = scope
    end

    def resolve
      raise NotImplementedError
    end

    private

    attr_reader :user, :scope
  end
end
```

### Simple Policy (Ownership + Admin)

```ruby
# app/policies/card_policy.rb
class CardPolicy < ApplicationPolicy
  def update?
    record.creator == user || user.admin?
  end

  def destroy?
    update?
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      scope.where(account: user.account)
    end
  end
end
```

### Role-Based Policy (RBAC)

```ruby
class User < ApplicationRecord
  enum :role, {regular: 0, admin: 1, librarian: 2}

  PERMISSIONS = {
    regular: %i[browse_catalogue borrow_books],
    librarian: %i[browse_catalogue borrow_books manage_books],
    admin: %i[browse_catalogue borrow_books manage_books manage_librarians]
  }.freeze

  def permission?(name)
    PERMISSIONS.fetch(role.to_sym, []).include?(name)
  end
end

# app/policies/book_policy.rb
class BookPolicy < ApplicationPolicy
  def show?
    true
  end

  def destroy?
    return true if user.permission?(:manage_all_books)
    return false unless user.permission?(:manage_books)

    # Attribute-based: department must match
    record.dept == user.dept
  end

  def create?
    user.permission?(:manage_books)
  end

  def update?
    create?
  end

  class Scope < ApplicationPolicy::Scope
    def resolve
      if user.permission?(:manage_all_books)
        scope.all
      elsif user.permission?(:manage_books)
        scope.where(dept: user.dept)
      else
        scope.none
      end
    end
  end
end
```

### Controller with `authorize` and `policy_scope`

```ruby
class BooksController < ApplicationController
  def index
    @books = policy_scope(Book)
  end

  def destroy
    @book = Book.find(params[:id])
    authorize @book

    @book.destroy!
    redirect_to books_path, notice: "Removed"
  end
end

class CardsController < ApplicationController
  def update
    @card = Card.find(params[:id])
    authorize @card

    @card.update!(card_params)
  end

  def index
    @cards = policy_scope(Card)
  end
end
```

### Authorization in Views

```erb
<% @books.each do |book| %>
  <li>
    <%= book.name %>
    <% if policy(book).destroy? %>
      <%= button_to "Delete", book, method: :delete %>
    <% end %>
  </li>
<% end %>
```

### N+1 Authorization Problem

```erb
<% @posts.each do |post| %>
  <% if policy(post).publish? %>  <%# N checks! %>
    <%= button_to "Publish", ... %>
  <% end %>
<% end %>
```

**Solutions:**
- Preload data needed by policy rules
- Cache authorization results
- Use scoping-based authorization (filter before rendering)

### Testing

Test policy rules separately from enforcement:

```ruby
class BookPolicyTest < ActiveSupport::TestCase
  def setup
    @librarian = users(:librarian)  # dept: "fiction"
    @book = books(:fiction_book)    # dept: "fiction"
  end

  test "allows librarians to destroy books in their department" do
    policy = BookPolicy.new(@librarian, @book)
    assert policy.destroy?
  end

  test "denies librarians from other departments" do
    other_book = books(:nonfiction_book)  # dept: "non-fiction"
    policy = BookPolicy.new(@librarian, other_book)
    assert_not policy.destroy?
  end

  test "allows admins to destroy any book" do
    admin = users(:admin)
    policy = BookPolicy.new(admin, @book)
    assert policy.destroy?
  end
end

class BookPolicyScopeTest < ActiveSupport::TestCase
  test "librarians see only their department" do
    librarian = users(:librarian)
    scope = BookPolicy::Scope.new(librarian, Book.all).resolve
    assert scope.all? { |b| b.dept == librarian.dept }
  end
end
```

Test enforcement in integration tests:

```ruby
class BooksControllerTest < ActionDispatch::IntegrationTest
  test "unauthorized user cannot destroy book" do
    sign_in users(:regular)
    book = books(:fiction_book)

    assert_no_difference "Book.count" do
      delete book_path(book)
    end

    assert_response :redirect
  end
end
```

### Anti-patterns

**Authorization in models:**

```ruby
# BAD
class Book < ApplicationRecord
  def destroyable_by?(user)
    user.admin? || user.dept == dept
  end
end

# GOOD: Keep in policy
class BookPolicy < ApplicationPolicy
  def destroy?
    # ...
  end
end
```

**Mixed enforcement layers:**

```ruby
# BAD: Authorization in both controller AND service
class BooksController
  def destroy
    authorize @book  # Here...
    BookService.destroy(@book, current_user)
  end
end

class BookService
  def destroy(book, user)
    raise unless user.can_destroy?(book)  # ...and here!
  end
end

# GOOD: Single enforcement point
class BooksController
  def destroy
    authorize @book
    @book.destroy!
  end
end
```

**When:** authorization logic is conditional, role-based, or duplicated across controllers.
**When not:** a simple `current_user.admin?` check in one place — inline is fine.

> See `shared/authorization.md` for the full authorization reference — layer
> placement, error handling (404 vs 403), ABAC patterns, and rule design heuristics.


## Calculator Objects

Extract complex calculations into a PORO namespaced under the model it serves.
Follows the `app/models/<model>/<concept>.rb` convention
(see `coding-classic.md § POROs Namespaced Under Models`).

```ruby
# app/models/line_item.rb
class LineItem < ApplicationRecord
  def price
    LineItems::Price.new(self).calculate
  end
end

# app/models/line_items/price.rb
module LineItems
  class Price
    def initialize(line_item)
      @line_item = line_item
      @product = line_item.product
    end

    def calculate
      base_price + options_price - discount
    end

    private

    def base_price
      @product.base_price
    end

    def options_price
      @line_item.options.sum(&:price)
    end

    def discount
      @line_item.coupon&.discount_amount || 0
    end
  end
end
```

**When:** a model method contains 3+ private helpers all dedicated to one calculation.
**When not:** the calculation is a simple one-liner — leave it on the model.

## Domain Models over Service Objects

**Key principle:** a representation of a business domain concept is called a **model**.
Instead of `*Service`, `*Manager`, `*Handler` — name the domain concept as a noun.

```ruby
# Bad — procedural, no domain identity
# app/services/notification_service.rb
class NotificationService
  def self.call(user, message)
    # sends notification
  end
end

# Good — domain model, lives in app/models/
# app/models/notification.rb
class Notification
  include ActiveModel::Model

  attr_accessor :user, :message

  def deliver
    # sends notification
  end
end
```

### Naming Guidelines

| Instead of | Use |
|------------|-----|
| `UserSignupService` | `Registration` or `UserSignup` |
| `PaymentProcessor` | `Payment` |
| `NotificationService` | `Notification` or `NotificationDelivery` |
| `EmailSender` | `Email` or `EmailMessage` |
| `OrderCreator` | `Order` or `OrderPlacement` |
| `InvitationManager` | `Invitation` |

**Rule**: think of the **noun** that describes what this thing *is*, not what it *does*.

These domain models use `ActiveModel::Model` for validation, form integration, and
mass assignment from params — the same foundation as Form Objects. For simple
multi-model orchestration that doesn't need the full `ApplicationForm` base class,
a plain `ActiveModel::Model` class with a `save` method is enough:

```ruby
# app/models/registration.rb — lightweight domain model
class Registration
  include ActiveModel::Model

  attr_accessor :email, :password, :company_name

  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }
  validates :company_name, presence: true

  def save
    return false unless valid?

    create_user
    create_company
    send_welcome_email
    true
  end

  private

  def create_user
    @user = User.create!(email: email, password: password)
  end

  def create_company
    @company = Company.create!(name: company_name, owner: @user)
  end

  def send_welcome_email
    RegistrationMailer.welcome(@user).deliver_later
  end
end
```

**When to reach for `ApplicationForm` instead:** when you need `after_save` callbacks,
`model_name` quacking for route helpers, or `submit!` / `with_transaction` plumbing.

See also: `refactorings/010-refactor-service-object-into-poro.md`,
`review-architecture.md § Anti-patterns > Service Objects`

## State Machines

When a model has multiple related boolean/timestamp attributes tracking state,
or complex conditional logic based on object state, extract into an explicit
state machine using AASM. Prefer events over direct state assignment
(`order.submit!` not `order.status = :submitted`).

For simple linear progressions, a plain `enum` is enough — don't reach for a
state machine prematurely.

> See `shared/state_machines.md` for the full reference — AASM setup, guards,
> callbacks, `after_commit` for side effects, standalone state machines, and
> triggering deliveries from transitions.

## Kredis UI State Container

Generalized pattern for persisting ephemeral UI state (accordion open/closed,
tab selections, sidebar collapse) across page reloads using Kredis, Stimulus,
and server-side DOM rehydration with Nokogiri. Works best with HTML-over-the-wire
architectures where UI state lives in HTML attributes.

### Key Generation

Generate obfuscated, user-scoped Kredis keys by combining the current user,
resource, and template location. Digest prevents tampering in dev tools:

```ruby
# app/helpers/application_helper.rb
def ui_state_key(name)
  key = cache_fragment_name(name, skip_digest: true)
    .flatten.compact
    .map(&:cache_key)
    .join(":")
  key += ":#{caller.find { _1 =~ /html/ }}"
  ActiveSupport::Digest.hexdigest(key)
end
```

### MutationObserver + Stimulus Controller

A Stimulus controller watches all attribute changes on its element via
`MutationObserver` and PATCHes them to a server endpoint. No per-feature
wiring needed — any attribute change is captured automatically:

```javascript
// app/javascript/controllers/ui_state_controller.js
export default class extends Controller {
  static values = { key: String };

  connect() {
    this.mutationObserver = new MutationObserver(
      this.mutateState.bind(this)
    );
    this.mutationObserver.observe(this.element, { attributes: true });
  }

  disconnect() {
    this.mutationObserver?.disconnect();
  }

  async mutateState() {
    const body = new FormData();
    const attributes = {};
    this.element.getAttributeNames()
      .filter((name) => name !== "data-controller")
      .map((name) => {
        attributes[name] = this.element.getAttribute(name);
      });
    body.append("key", this.keyValue);
    body.append("attributes", JSON.stringify(attributes));
    await patch("/ui_state/update", { body });
  }
}
```

### Server-Side Storage

Store sanitized attributes in Kredis as JSON:

```ruby
class UiStateController < ApplicationController
  def update
    ui_state = Kredis.json params[:key]
    ui_state.value = JSON.parse(params[:attributes])
      .deep_transform_values { sanitize(_1) }
    head :ok
  end
end
```

### Server-Side Rehydration with Nokogiri

On render, restore saved attributes onto the DOM element before it reaches the
browser. Nokogiri parses the block's HTML, applies stored attributes, and
injects the Stimulus controller:

```ruby
# app/helpers/application_helper.rb
def remember_ui_state_for(name, &block)
  html = capture(&block).to_s.strip
  fragment = Nokogiri::HTML.fragment(html)
  el = fragment.first_element_child

  ui_state = Kredis.json(ui_state_key(name))
  ui_state.value&.each do |attr, value|
    el[attr] = sanitize(value, tags: [])
  end

  el["data-controller"] =
    "#{el["data-controller"]} ui-state".strip
  el["data-ui-state-key-value"] = ui_state_key(name)

  el.to_html.html_safe
end
```

Usage in views:

```erb
<%= remember_ui_state_for([current_user, product]) do %>
  <details>
    <summary>Description</summary>
    <%= product.description %>
  </details>
<% end %>
```

**When to reach for this:** you need per-user UI state persistence (accordions,
collapsible panels, tab selections) without building individual Kredis keys for
each feature. One generalized mechanism covers any element whose state lives in
HTML attributes.

## DCI (Data, Context, Interaction) — Supplement

> Core pattern covered in `review-architecture.md § DCI`. This section adds
> the `surrounded` gem approach and testing guidance from practical use.

### The `surrounded` Gem

Jim Gay's `surrounded` gem reduces DCI boilerplate. Define roles inline with
`role` blocks — methods become available only within the context:

```ruby
class Checkout
  extend Surrounded::Context

  role :payable do
    def create_session
      Stripe::Checkout::Session.create({
        line_items: line_items,
        metadata: { gid: to_gid.to_s },
        success_url: polymorphic_url(self)
      })
    end

    def fulfill
      # fulfillment logic
    end
  end
end

# Usage — same context works for any object with `line_items`
Checkout.new(payable: @quote).create_session
Checkout.new(payable: @review).create_session
```

Roles are garbage-collected with the context — no permanent pollution of the
data model.

### Testing DCI Contexts

Test through the context directly. Enact the interaction and verify outcomes:

```ruby
class CheckoutTest < ActiveSupport::TestCase
  test "stripe session includes gid in metadata" do
    quote = quotes(:accepted_quote)
    session = Checkout.new(payable: quote).create_session
    assert_equal quote, GlobalID::Locator.locate(session.metadata.gid)
  end
end
```

### When DCI Fits Best

Best for third-party API integrations and fringe concerns that don't change
often. Avoid for core domain logic where the behavioral split may reduce
cohesion. The context should make no assumptions about concrete types — only
that mapped roles respond to the required interface (`line_items`, `increase`,
`decrease`, etc.).

## ActiveStorage Custom Analyzers

Extend ActiveStorage's analysis pipeline with custom analyzers that extract
domain-specific metadata at upload time. Metadata persists in the
`active_storage_blobs.metadata` JSON column — computed once, available everywhere.

### Custom Analyzer Class

Subclass `ActiveStorage::Analyzer` (or a built-in like `AudioAnalyzer`,
`ImageAnalyzer::ImageMagick`). Override `metadata` and merge with `super`:

```ruby
# lib/active_storage/waveform_analyzer.rb
module ActiveStorage
  class WaveformAnalyzer < ActiveStorage::Analyzer::AudioAnalyzer
    def metadata
      super.merge(waveform)
    end

    private

    def waveform
      rms_values = []
      download_blob_to_tempfile do |file|
        IO.popen([ffmpeg_path, "-i", file.path, "-ac", "1",
                  "-f", "f32le", "-"], "rb") do |io|
          chunk_size = 512 * 4
          while (chunk = io.read(chunk_size))
            floats = chunk.unpack("e*")
            next if floats.empty?
            rms = Math.sqrt(floats.sum { _1**2 } / floats.size)
            rms_values << rms
          end
        end
      end
      { waveform: rms_values }
    end

    def ffmpeg_path
      ActiveStorage.paths[:ffmpeg] || "ffmpeg"
    end
  end
end
```

### Blurhash Analyzer

```ruby
class BlurhashAnalyzer < ActiveStorage::Analyzer::ImageAnalyzer::ImageMagick
  def metadata
    read_image do |image|
      build_thumbnail(image)
      super.merge(blurhash: compute_blurhash)
    end
  end

  private

  def build_thumbnail(image)
    @thumbnail ||= MiniMagick::Image.open(
      ImageProcessing::MiniMagick
        .source(image.path)
        .resize_to_limit(200, 200)
        .loader(page: 0)
        .call.path
    )
  end

  def compute_blurhash
    Blurhash.encode(@thumbnail.width, @thumbnail.height,
                    @thumbnail.get_pixels.flatten)
  end
end
```

### Registration Order

Register in an initializer with `prepend` — the **first** analyzer whose
`accept?` returns `true` wins. `prepend` ensures your custom analyzer runs
before built-in ones:

```ruby
# config/initializers/active_storage.rb
require_relative "../../lib/active_storage/waveform_analyzer"

Rails.application.config.active_storage.analyzers.prepend(
  ActiveStorage::WaveformAnalyzer
)
```

### Accessing Metadata

No migration needed — metadata lives in the existing `metadata` text column:

```ruby
song.recording.blob.metadata
# => {"identified"=>true, "waveform"=>[0.000123, 0.000873, ...]}

song.recording.metadata[:waveform]
```

**When to reach for this:** you need to precompute expensive presentation data
(waveforms for audio players, blurhashes for image placeholders, PDF page
counts) once at upload time rather than on every render.

## ActiveStorage Custom Previewers

Convert non-image attachments (audio, video, PDFs) into displayable image
previews. Previewers complement analyzers — analyzers extract metadata,
previewers generate visual representations.

### Custom Previewer Class

Subclass `ActiveStorage::Previewer`. Implement the class method `accept?` and
the instance method `preview`:

```ruby
# lib/active_storage/waveform_previewer.rb
require "chunky_png"

module ActiveStorage
  class WaveformPreviewer < ActiveStorage::Previewer
    class << self
      def accept?(blob)
        blob.audio? && blob.metadata[:waveform].present?
      end
    end

    def preview(**options)
      waveform = blob.metadata.fetch(:waveform, [])
      width, height = 640, 240
      center = height / 2

      png = ChunkyPNG::Image.new(width, height, ChunkyPNG::Color::WHITE)
      frame_width = waveform.size / width

      waveform.each_with_index do |v, idx|
        next unless idx % frame_width == 0
        x = idx / frame_width
        y_val = (v * center).to_i
        png.rect(x, center - y_val, x, center + y_val,
                 0x4b0082ff, 0x4b0082ff)
      end

      io = StringIO.new
      png.write(io)
      io.rewind

      yield io: io,
            filename: "#{blob.filename.base}.png",
            content_type: "image/png",
            **options
    end
  end
end
```

### Registration

Same pattern as analyzers — `prepend` to take priority over built-in
previewers:

```ruby
# config/initializers/active_storage.rb
Rails.application.config.active_storage.previewers.prepend(
  ActiveStorage::WaveformPreviewer
)
```

### Display in Templates

Call `preview` like you would `variant`:

```erb
<%= image_tag song.recording.preview(resize_to_fill: [640, 160]) %>
```

**When to reach for this:**

- **Previewers** — non-image media needs a visual representation (audio
  waveform PNG, video thumbnail, PDF first-page image).
- **Analyzers** — you need metadata extracted, not a visual preview.
