---
name: jr-rails-phlex
description: >-
  Write Phlex views and components for Rails: class hierarchy, slots, helpers,
  custom elements, scaffold generator. Use when building UI with phlex-rails.
---

# Phlex Views & Components

Components are Ruby objects — no template language, no DSL. Views replace ERB
templates (one per controller action). Components are reusable UI building blocks.

## Core Workflow

1. **Use the scaffold generator** — custom `PhlexControllerGenerator` produces
   Phlex views instead of ERB. See [reference/coding-phlex.md § Scaffolding](reference/coding-phlex.md) for the full generator code and all 5 templates.
2. **Implementation order** — models → controllers → views/components → tests.
3. **Forms default to ERB partials** — Rails form builders are ergonomic there.
   Phlex form helpers exist for simple cases.

## Class Hierarchy

```
Components::Base < Phlex::HTML   (include Components, route helpers, dev comments)
  ├── Views::Base                (+ Debug, ContentFor, caching)
  ├── Components::Layout         (+ Phlex::Rails::Layout)
  └── Components::PageHeader, TitleBar, List, GridCell, ...
```

```ruby
# app/components/base.rb
class Components::Base < Phlex::HTML
  include Components
  include Phlex::Rails::Helpers::Routes

  if Rails.env.development?
    def before_template
      comment { "Before #{self.class.name}" }
      super
    end
  end
end

# app/views/base.rb
class Views::Base < Components::Base
  include Phlex::Rails::Helpers::Debug

  def cache_store = Rails.cache
end
```

## Short-form Component Calls

`include Components` on the base class enables method-style rendering:

```ruby
# Instead of:
render Components::PageHeader.new(title: "Labels")

# Write:
PageHeader(title: "Labels")
```

## Component Slots via Methods

Public methods on components yield named content areas:

```ruby
class Components::TitleBar < Components::Base
  def view_template(&) = div(class: "title-bar", &)
  def leading_action(&) = div(class: "leading-action", &)
  def title(&) = h1(&)
  def actions(&) = div(class: "actions", &)
end

# Usage:
render Components::TitleBar.new do |bar|
  bar.leading_action { link_to("Back", labels_path) }
  bar.title { "My Page" }
  bar.actions { button_to("Delete", @label, method: :delete) }
end
```

## Helper Includes

Include only the `Phlex::Rails::Helpers::*` each view needs — never on Base:

```ruby
class Views::Labels::Show < Views::Base
  include Phlex::Rails::Helpers::ContentFor
  include Phlex::Rails::Helpers::DOMID
  include Phlex::Rails::Helpers::ButtonTo
  include Phlex::Rails::Helpers::LinkTo
  include Phlex::Rails::Helpers::TurboFrameTag
  # ...
end
```

Common helpers: `Routes`, `ContentFor`, `DOMID`, `ButtonTo`, `LinkTo`,
`TurboFrameTag`, `TurboStreamFrom`, `ImageTag`, `ClassNames`, `Request`,
`Notice`, `Debug`, `Sanitize`, `StripTags`.

## Content Areas

Layout yields named areas; views populate via `content_for`:

```ruby
content_for :title, "Labels"

content_for :main_header do
  render Components::PageHeader.new do |header|
    header.title_bar { |bar| bar.title { "Labels" } }
  end
end

content_for :floating_action do
  render Components::FloatingActionMenu.new
end
```

Standard areas: `:title`, `:main_header`, `:floating_action`, `:head`.

## Controller Rendering

Controllers render Phlex views directly, passing data via `new`:

```ruby
class LabelsController < ApplicationController
  def index
    @pagy, @labels = pagy(Label.all)
    render Views::Labels::Index.new(@labels, @pagy)
  end

  def show
    render Views::Labels::Show.new(@label)
  end

  def create
    @label = Label.new(label_params)
    if @label.save
      redirect_to @label, notice: "Label was successfully created."
    else
      render Views::Labels::New.new(@label), status: :unprocessable_entity
    end
  end
end
```

## Custom Element Wrappers

Use `register_element` for web component custom elements (Web Awesome, Shoelace, etc.):

```ruby
module Components::MyLibrary
  class MyButton < Phlex::HTML
    register_element :my_button  # renders <my-button>

    def initialize(variant: "neutral", size: "medium", **attributes)
      @attributes = attributes.with_defaults(variant: variant, size: size)
    end

    def view_template(&) = my_button(**@attributes, &)
  end
end
```

Use [`phlex_custom_element_generator`](https://github.com/konnorrogers/phlex_custom_element_generator) to auto-generate wrappers from custom element manifests.

## Register Helpers

For Rails helpers that output HTML or return values:

```ruby
register_output_helper :vite_client_tag       # returns HTML
register_output_helper :vite_javascript_tag
register_output_helper :column_chart          # chartkick
register_value_helper :alert                  # returns a value
```

## ERB Partials for Forms

Phlex views render ERB form partials seamlessly:

```ruby
# In a Phlex view:
section { render partial("form", label: @label) }

# The ERB partial uses standard form_with / form_for
```

## Fragment Caching

```ruby
def view_template
  cache("labels/#{@label.id}/card") {
    # expensive rendering
  }
end
```

## Multiple Layouts

```ruby
class Components::Layout < Components::Base
  include Phlex::Rails::Layout
  # app shell
end

class Components::MarketingLayout < Components::Base
  include Phlex::Rails::Layout
  # landing pages
end
```

Set in controllers: `layout -> { Components::Layout }`

## Frontend Integration

### Stimulus

```ruby
div(data: {
  controller: "faceted-search",
  action: "input->faceted-search#perform:prevent",
  faceted_search_url_value: labels_path
}) { ... }
```

### Turbo Frames (Lazy Loading)

```ruby
turbo_frame_tag(album, src: album_path(album), loading: :lazy) {
  render Components::Spinner.new
}
```

### Turbo Streams & Morphing

```ruby
turbo_stream_from([@budget, :items])

# In layout <head>:
meta name: "turbo-refresh-method", content: "morph"
meta name: "turbo-refresh-scroll", content: "preserve"
```

### Pagy

```ruby
class Views::Labels::Index < Views::Base
  include Pagy::Frontend

  def initialize(labels, pagy)
    @labels = labels
    @pagy = pagy
  end

  def view_template
    # ... render labels ...
    raw safe(pagy_nav(@pagy))
  end
end
```

## Heuristics

- **Component vs View:** reusable UI = component; page-level (one per action) = view
- **When to use ERB:** forms with Rails form builder, complex form logic
- **Naming:** `Components::PageHeader`, `Views::Labels::Index`
- **One view per controller action;** views receive data via `initialize`
- **Composition over inheritance** — slot methods, not deep class hierarchies
- **Data flow:** controller → view (`initialize`) → components (`render`)

## Anti-patterns

- **God components** — split into smaller, focused components
- **Business logic in views/components** — belongs in models
- **Over-including helpers on Base** — include per-view
- **Reimplementing form builders** — use ERB partials for forms
- **Passing request context into components** — components receive data, not request objects

## Preferred Stack

| Concern | Choice |
|---------|--------|
| Components | `phlex-rails` 2.x |
| Bundling | `vite_rails` / importmap / esbuild |
| Pagination | Pagy |
| Custom elements | library of choice + `phlex_custom_element_generator` |
| Scaffolding | Custom `PhlexControllerGenerator` |
| Frontend | `turbo-rails` + `stimulus-rails` |
| Feature flags | Flipper (inline: `if Flipper.enabled?(:feature)`) |

## Deep Reference Files

Read these on demand when the task requires deeper guidance:

- **Full Phlex coding guide** (generator templates, all examples): [reference/coding-phlex.md](reference/coding-phlex.md)
- **Design patterns** (form objects, query objects, strategies): [reference/patterns.md](reference/patterns.md)
- **Testing guide**: [reference/shared/testing.md](reference/shared/testing.md)
- **Hotwire** (Turbo + Stimulus): [reference/shared/hotwire.md](reference/shared/hotwire.md)
- **Components reference**: [reference/shared/components.md](reference/shared/components.md)
- **Architecture layers**: [reference/shared/architecture.md](reference/shared/architecture.md)

For frontend patterns (Stimulus controllers, Turbo Frames/Streams), invoke
the relevant `hwc-*` skill alongside this one.
