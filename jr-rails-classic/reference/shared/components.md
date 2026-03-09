# Components

## Summary

Components are Ruby objects that encapsulate view logic, replacing complex partials with testable, reusable units. They bridge the presentation layer's need for logic with proper object-oriented design. Phlex components render HTML via pure Ruby methods — no template language, no DSL.

> See `coding-phlex.md` for the full Phlex architecture guide — class hierarchy,
> generator scaffolding, content areas, custom element wrappers, and Turbo/Stimulus integration.

## Layer Placement

```
┌─────────────────────────────────────────┐
│ Presentation Layer                      │
│  ├─ Views (page-level, one per action)  │
│  ├─ Components (reusable UI building    │
│  │   blocks)                            │
│  └─ Presenters (view-specific logic)    │
└─────────────────────────────────────────┘
```

## When to Use

- Partials with complex logic
- Reusable UI elements across views
- Components needing isolated testing
- Encapsulating Stimulus controller wiring

## When NOT to Use

- Simple static markup (inline in view)
- One-off display logic (method on the view)
- Data transformation (use presenters/serializers)

## Key Principles

- **Single responsibility** — one component, one purpose
- **Explicit interface** — typed `initialize`, no magic
- **Isolated testing** — unit test without full request
- **Composition** — build complex UIs from simple components
- **Views receive data** — pass everything via `initialize`, no data fetching

## Implementation

### Base Class

```ruby
# app/components/base.rb
module Components
  class Base < Phlex::HTML
    include Components
    include Phlex::Rails::Helpers::Routes

    if Rails.env.development?
      def before_template
        comment { "#{self.class.name}" }
        super
      end
    end
  end
end
```

### Basic Component

```ruby
# app/components/user_avatar.rb
module Components
  class UserAvatar < Base
    SIZES = { small: 32, medium: 64, large: 128 }.freeze

    def initialize(user:, size: :medium)
      @user = user
      @size = size
    end

    def view_template
      if @user.avatar.attached?
        img(
          src: url_for(@user.avatar.variant(resize_to_limit: dimensions)),
          class: css_class,
          alt: @user.name
        )
      else
        div(class: "#{css_class} avatar-placeholder") { initials }
      end
    end

    private

    def initials
      @user.name.split.map(&:first).join.upcase[0, 2]
    end

    def dimensions
      size = SIZES.fetch(@size)
      [size, size]
    end

    def css_class
      "avatar avatar-#{@size}"
    end
  end
end
```

### Component with Slots (Content Methods)

In Phlex, slots are just public methods that accept blocks:

```ruby
# app/components/card.rb
module Components
  class Card < Base
    def initialize(variant: :default)
      @variant = variant
    end

    def view_template(&block)
      article(class: "card card-#{@variant}") do
        yield_content(&block)
      end
    end

    def header(&block)
      header_tag(class: "card-header", &block)
    end

    def body(&block)
      div(class: "card-body", &block)
    end

    def actions(&block)
      div(class: "card-actions", &block)
    end

    def footer(&block)
      footer(class: "card-footer", &block)
    end
  end
end
```

### Usage in Views

```ruby
# app/views/posts/show.rb
module Views
  module Posts
    class Show < Views::Base
      def initialize(post:)
        @post = post
      end

      def view_template
        Card(variant: :primary) do |card|
          card.header { "Post Details" }

          card.body do
            h2 { @post.title }
            p { @post.excerpt }
          end

          card.actions do
            a(href: helpers.post_path(@post), class: "btn") { "Read more" }
            a(href: helpers.share_path(@post), class: "btn btn-secondary") { "Share" }
          end

          card.footer { "Published #{helpers.time_ago_in_words(@post.published_at)} ago" }
        end
      end
    end
  end
end
```

### Component with Variants

```ruby
# app/components/alert.rb
module Components
  class Alert < Base
    VARIANTS = {
      info: { icon: "info-circle", class: "alert-info" },
      success: { icon: "check-circle", class: "alert-success" },
      warning: { icon: "exclamation-triangle", class: "alert-warning" },
      error: { icon: "x-circle", class: "alert-error" }
    }.freeze

    def initialize(variant: :info, dismissible: false)
      @variant = variant
      @dismissible = dismissible
      @config = VARIANTS.fetch(variant)
    end

    def view_template(&block)
      div(
        class: "alert #{@config[:class]}",
        **(@dismissible ? { data: { controller: "dismissible" } } : {})
      ) do
        span(class: "alert-icon") { @config[:icon] }
        div(class: "alert-content", &block)
        dismiss_button if @dismissible
      end
    end

    private

    def dismiss_button
      button(
        class: "alert-dismiss",
        data: { action: "dismissible#dismiss" }
      ) { "×" }
    end
  end
end
```

### Component Composition

```ruby
# app/components/post_list.rb
module Components
  class PostList < Base
    def initialize(posts:)
      @posts = posts
    end

    def view_template
      div(class: "post-list") do
        @posts.each do |post|
          render PostCard.new(post: post)
        end
      end
    end
  end
end
```

### Stimulus Integration

```ruby
# app/components/dropdown.rb
module Components
  class Dropdown < Base
    def initialize(label:)
      @label = label
    end

    def view_template(&block)
      div(data: { controller: "dropdown" }) do
        button(data: { action: "dropdown#toggle" }) { @label }
        div(data: { dropdown_target: "menu" }, class: "hidden", &block)
      end
    end
  end
end
```

### Short-Form Component Calls

Include `Components` in your base to enable `Alert(variant: :success)` instead
of `render Components::Alert.new(variant: :success)`:

```ruby
module Components
  # For each component Foo, defines a method Foo(**args, &block)
  # that calls render(Components::Foo.new(**args), &block)
end
```

See `coding-phlex.md § Short-form Component Calls` for the implementation.

## Testing Components

```ruby
class Components::UserAvatarTest < ActiveSupport::TestCase
  include Phlex::Testing::ViewHelper

  test "renders avatar image when attached" do
    user = users(:david)
    # Assume fixture has avatar attached

    html = render(Components::UserAvatar.new(user: user))

    assert_includes html, "img"
    assert_includes html, "avatar"
  end

  test "renders initials when no avatar" do
    user = User.new(name: "John Doe")

    html = render(Components::UserAvatar.new(user: user))

    assert_includes html, "JD"
    assert_includes html, "avatar-placeholder"
  end

  test "applies size class" do
    user = User.new(name: "Jane")

    html = render(Components::UserAvatar.new(user: user, size: :large))

    assert_includes html, "avatar-large"
  end
end

class Components::CardTest < ActiveSupport::TestCase
  include Phlex::Testing::ViewHelper

  test "renders card with header and body" do
    html = render(Components::Card.new(variant: :primary)) do |card|
      card.header { "Title" }
      card.body { "Content" }
    end

    assert_includes html, "card-primary"
    assert_includes html, "Title"
    assert_includes html, "Content"
  end
end
```

## Extraction Signals

### From Helpers

Extract helpers to components when you see:

| Signal | Example | Action |
|--------|---------|--------|
| Heavy `tag.*` usage | `tag.div`, `tag.button` chains | Extract to component |
| Complex data attributes | `data: { controller: ..., action: ... }` | Component encapsulates Stimulus wiring |
| Conditional CSS classes | `class: "foo #{bar if baz}"` | Component method for class logic |
| Nested structure | Helper yielding to blocks with wrappers | Component with slot methods |

### From Presenters

Extract presenters to components when:
- Presenter has a `render` method returning HTML
- Presenter accepts `context: self` for helper access
- Presenter builds complex markup

**Signal:** `SomePresentation.new(..., context: self).render` — this indicates the
presenter is doing component work without component benefits.

## Anti-Patterns

### Data Fetching in Components

```ruby
# BAD: Component fetches its own data
module Components
  class RecentPosts < Base
    def view_template
      Post.recent.limit(5).each do |post|
        render PostCard.new(post: post)
      end
    end
  end
end

# GOOD: Data passed via initialize
module Components
  class RecentPosts < Base
    def initialize(posts:)
      @posts = posts
    end

    def view_template
      @posts.each { |post| render PostCard.new(post: post) }
    end
  end
end
```

### Business Logic in Components

```ruby
# BAD: Authorization in component
module Components
  class PostActions < Base
    def initialize(post:, user:)
      @post = post
      @user = user
    end

    def view_template
      a(href: helpers.edit_post_path(@post)) { "Edit" } if @user.admin? || @post.author == @user
    end
  end
end

# GOOD: Receive authorization result
module Components
  class PostActions < Base
    def initialize(post:, can_edit:, can_delete:)
      @post = post
      @can_edit = can_edit
      @can_delete = can_delete
    end

    def view_template
      a(href: helpers.edit_post_path(@post)) { "Edit" } if @can_edit
      button(formaction: helpers.post_path(@post)) { "Delete" } if @can_delete
    end
  end
end
```

### God Components

```ruby
# BAD: Component does too much
module Components
  class Post < Base
    # Renders post, comments, author, likes, shares, related posts...
  end
end

# GOOD: Compose from smaller components
def view_template
  render PostHeader.new(post: @post)
  render PostBody.new(post: @post)
  render PostActions.new(post: @post, can_edit: @can_edit, can_delete: @can_delete)
  render CommentsList.new(comments: @post.comments)
end
```

## File Organization

```
app/
├── components/
│   ├── base.rb
│   ├── alert.rb
│   ├── card.rb
│   ├── dropdown.rb
│   ├── user_avatar.rb
│   └── posts/
│       ├── card.rb
│       └── list.rb
└── views/
    ├── base.rb
    └── posts/
        ├── index.rb
        ├── show.rb
        ├── new.rb
        └── edit.rb
```

## Component vs View Heuristic

| | Component | View |
|--|-----------|------|
| **Purpose** | Reusable UI building block | Page-level, one per controller action |
| **Data** | Receives via `initialize` | Receives via `initialize` from controller |
| **Namespace** | `Components::Card` | `Views::Posts::Show` |
| **Reuse** | Used across multiple views | One-to-one with controller action |
| **Layout** | No layout | Renders inside layout |

**When to use ERB instead:** Forms — the Rails form builder (`form_with`,
field helpers, error display) works natively in ERB. Render ERB form partials
from Phlex views via `render partial("form", post: @post)`. See
`coding-phlex.md § ERB Partials for Forms`.
