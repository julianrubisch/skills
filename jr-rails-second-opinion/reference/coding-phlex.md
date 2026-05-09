# Coding Style: Modern Phlex Frontend

## Philosophy

**Components are Ruby objects — no template language, no DSL:**
- Views replace ERB templates, one per controller action
- Components are reusable UI building blocks (buttons, cards, headers)
- Views are page-level compositions of components
- `initialize` receives data; `view_template` renders HTML
- Override `rails g scaffold` to produce Phlex views instead of ERB

**ERB for forms, Phlex for everything else:**
- Forms default to ERB partials — Rails form builders are ergonomic there
- Phlex views render form partials via `render partial("form", thing: @thing)`
- Simple forms *can* use Phlex form helpers, but ERB is the pragmatic default

**Include only what you need:**
- Each view/component includes only the `Phlex::Rails::Helpers::*` it uses
- No god-mode base class with every helper pre-included

---

## Scaffolding / Generator

A custom scaffold generator replaces Rails' default ERB generation with Phlex views.

### Generator

Place at `lib/generators/rails/phlex_controller_generator.rb`:

```ruby
require "rails/generators"
require "rails/generators/actions"
require "rails/generators/named_base"
require "rails/generators/rails/scaffold_controller/scaffold_controller_generator"

module Rails
  module Generators
    class PhlexControllerGenerator < ScaffoldControllerGenerator
      source_root File.expand_path("templates", __dir__)

      def create_view_files
        template "index.rb", File.join("app/views", view_files_path, "index.rb")
        template "show.rb", File.join("app/views", view_files_path, "show.rb")
        template "new.rb", File.join("app/views", view_files_path, "new.rb")
        template "edit.rb", File.join("app/views", view_files_path, "edit.rb")
      end

      def remove_erb_files
        remove_file File.join("app/views", view_files_path, "index.html.erb")
        remove_file File.join("app/views", view_files_path, "show.html.erb")
        remove_file File.join("app/views", view_files_path, "new.html.erb")
        remove_file File.join("app/views", view_files_path, "edit.html.erb")
      end

      private

      def view_files_path
        !namespaced? ? plural_name : (namespace_dirs + [plural_name])
      end
    end
  end
end
```

### Templates

All templates go in `lib/generators/rails/templates/`.

#### `controller.rb.tt`

```erb
<%% module_namespacing do -%>
class <%%= controller_class_name %>Controller < ApplicationController
  before_action :set_<%%= singular_table_name %>, only: %i[ show edit update destroy ]

  # GET <%%= route_url %>
  def index
    @<%%= plural_table_name %> = <%%= orm_class.all(class_name) %>

    render Views::<%%= singular_table_name.classify.pluralize %>::Index.new(@<%%= plural_table_name %>)
  end

  # GET <%%= route_url %>/1
  def show
    render Views::<%%= singular_table_name.classify.pluralize %>::Show.new(@<%%= singular_table_name %>)
  end

  # GET <%%= route_url %>/new
  def new
    @<%%= singular_table_name %> = <%%= orm_class.build(class_name) %>

    render Views::<%%= singular_table_name.classify.pluralize %>::New.new(@<%%= singular_table_name %>)
  end

  # GET <%%= route_url %>/1/edit
  def edit
    render Views::<%%= singular_table_name.classify.pluralize %>::Edit.new(@<%%= singular_table_name %>)
  end

  # POST <%%= route_url %>
  def create
    @<%%= singular_table_name %> = <%%= orm_class.build(class_name, "#{singular_table_name}_params") %>

    if @<%%= orm_instance.save %>
      redirect_to <%%= redirect_resource_name %>, notice: <%%= %("#{human_name} was successfully created.") %>
    else
      render Views::<%%= singular_table_name.classify.pluralize %>::New.new(@<%%= singular_table_name %>), status: :unprocessable_entity
    end
  end

  # PATCH/PUT <%%= route_url %>/1
  def update
    if @<%%= orm_instance.update("#{singular_table_name}_params") %>
      redirect_to <%%= redirect_resource_name %>, notice: <%%= %("#{human_name} was successfully updated.") %>, status: :see_other
    else
      render Views::<%%= singular_table_name.classify.pluralize %>::Edit.new(@<%%= singular_table_name %>), status: :unprocessable_entity
    end
  end

  # DELETE <%%= route_url %>/1
  def destroy
    @<%%= orm_instance.destroy %>
    redirect_to <%%= index_helper %>_path, notice: <%%= %("#{human_name} was successfully destroyed.") %>, status: :see_other
  end

  private
    def set_<%%= singular_table_name %>
      @<%%= singular_table_name %> = <%%= orm_class.find(class_name, "params.expect(:id)") %>
    end

    def <%%= "#{singular_table_name}_params" %>
      <%%- if attributes_names.empty? -%>
      params.fetch(:<%%= singular_table_name %>, {})
      <%%- else -%>
      params.expect(<%%= singular_table_name %>: [ <%%= permitted_params %> ])
      <%%- end -%>
    end
end
<%% end -%>
```

#### `index.rb.tt`

```erb
# frozen_string_literal: true

class Views::<%%= class_name.pluralize %>::Index < Views::Base
  include Phlex::Rails::Helpers::Routes
  include Phlex::Rails::Helpers::ContentFor
  include Phlex::Rails::Helpers::DOMID
  include Phlex::Rails::Helpers::ButtonTo

  def initialize(<%%= model_resource_name.pluralize %>)
    <%%= model_resource_name(prefix: "@").pluralize %> = <%%= model_resource_name.pluralize %>
  end

  def view_template
    content_for :title, "<%%= human_name.pluralize %>"

    content_for :main_header do
      render Components::PageHeader.new do |header|
        header.title_bar { |title_bar|
          title_bar.title {
            "<%%= human_name.pluralize %>"
          }

          title_bar.actions {
            # add "New" button here
          }
        }
      end
    end

    section {
      @<%%= plural_table_name %>.each do |<%%= singular_table_name %>|
        div(id: dom_id(<%%= singular_table_name %>)) {
          # add item rendering here
        }
      end
    }
  end
end
```

#### `show.rb.tt`

```erb
# frozen_string_literal: true

class Views::<%%= class_name.pluralize %>::Show < Views::Base
  include Phlex::Rails::Helpers::Routes
  include Phlex::Rails::Helpers::ContentFor
  include Phlex::Rails::Helpers::DOMID
  include Phlex::Rails::Helpers::ButtonTo

  def initialize(<%%= model_resource_name %>)
    <%%= model_resource_name(prefix: "@") %> = <%%= model_resource_name %>
  end

  def view_template
    content_for :title, "<%%= human_name %>"

    content_for :main_header do
      render Components::PageHeader.new do |header|
        header.title_bar { |title_bar|
          title_bar.leading_action {
            # add back button here
          }

          title_bar.title {
            # add title
          }

          title_bar.actions {
            # add edit/delete buttons here
          }
        }
      end
    end

    section {
      # add body here
    }
  end
end
```

#### `new.rb.tt`

```erb
# frozen_string_literal: true

class Views::<%%= class_name.pluralize %>::New < Views::Base
  include Phlex::Rails::Helpers::Routes
  include Phlex::Rails::Helpers::ContentFor
  include Phlex::Rails::Helpers::DOMID

  def initialize(<%%= model_resource_name %>)
    <%%= model_resource_name(prefix: "@") %> = <%%= model_resource_name %>
  end

  def view_template
    content_for :title, "New <%%= human_name.downcase %>"

    content_for :main_header do
      render Components::PageHeader.new do |header|
        header.title_bar { |title_bar|
          title_bar.leading_action {
            # add back button here
          }

          title_bar.title { "New <%%= human_name.downcase %>" }
        }
      end
    end

    section {
      render partial("form", <%%= model_resource_name %>: <%%= model_resource_name(prefix: "@") %>)
    }
  end
end
```

#### `edit.rb.tt`

```erb
# frozen_string_literal: true

class Views::<%%= class_name.pluralize %>::Edit < Views::Base
  include Phlex::Rails::Helpers::Routes
  include Phlex::Rails::Helpers::ContentFor
  include Phlex::Rails::Helpers::DOMID

  def initialize(<%%= model_resource_name %>)
    <%%= model_resource_name(prefix: "@") %> = <%%= model_resource_name %>
  end

  def view_template
    content_for :title, "Editing <%%= human_name.downcase %>"

    content_for :main_header do
      render Components::PageHeader.new do |header|
        header.title_bar { |title_bar|
          title_bar.leading_action {
            # add back button here
          }

          title_bar.title { "Editing <%%= human_name.downcase %>" }
        }
      end
    end

    section {
      render partial("form", <%%= model_resource_name %>: <%%= model_resource_name(prefix: "@") %>)
    }
  end
end
```

---

## Patterns

### Class Hierarchy

```
Components::Base < Phlex::HTML
├── include Components            (short-form component calls)
├── include Phlex::Rails::Helpers::Routes
├── dev-only before_template comment
│
├── Views::Base < Components::Base
│   ├── include Phlex::Rails::Helpers::Debug
│   ├── include Phlex::Rails::Helpers::ContentFor  (optional, per-project)
│   └── cache_store → Rails.cache
│
├── Components::Layout < Components::Base
│   ├── include Phlex::Rails::Layout
│   ├── include Phlex::Rails::Helpers::ContentFor
│   ├── include Phlex::Rails::Helpers::Notice
│   ├── register_output_helper :vite_client_tag
│   ├── register_output_helper :vite_javascript_tag
│   ├── register_value_helper :alert
│   └── yields :main_header, :floating_action, :head
│
└── Components::PageHeader, TitleBar, Grid, GridCell, List, ListItem, ...
```

#### Base classes

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
```

```ruby
# app/views/base.rb
class Views::Base < Components::Base
  include Phlex::Rails::Helpers::Debug

  def cache_store
    Rails.cache
  end
end
```

```ruby
# app/components/layout.rb
class Components::Layout < Components::Base
  include Phlex::Rails::Layout
  include Phlex::Rails::Helpers::ContentFor
  include Phlex::Rails::Helpers::Notice

  register_output_helper :vite_client_tag
  register_output_helper :vite_javascript_tag
  register_value_helper :alert

  def view_template(&block)
    doctype
    html(lang: "en") do
      head do
        title { content_for(:title) || Rails.application.name }
        meta name: "viewport", content: "width=device-width,initial-scale=1"
        meta name: "turbo-refresh-method", content: "morph"
        meta name: "turbo-refresh-scroll", content: "preserve"

        csp_meta_tag
        csrf_meta_tags

        vite_client_tag
        vite_javascript_tag "application"

        raw safe(content_for(:head).to_s) if content_for?(:head)
      end

      body do
        # app shell: header, notice/alert, main content, footer
        header { render SiteHeader.new }

        if notice.present?
          # flash notice rendering
        end

        yield :main_header
        main(&block)

        if content_for?(:floating_action)
          div(class: "floating-action-menu") { yield :floating_action }
        end

        footer { # footer content }
      end
    end
  end
end
```

### Component Slots via Methods

Components expose named content areas as public methods. The caller uses block syntax to fill them:

```ruby
# app/components/title_bar.rb
class Components::TitleBar < Components::Base
  def view_template(&)
    div(class: "title-bar", &)
  end

  def leading_action(&)
    div(class: "leading-action", &)
  end

  def title(&)
    h1(&)
  end

  def trailing_visual(&)
    div(&)
  end

  def actions(&)
    div(class: "actions", &)
  end
end
```

```ruby
# app/components/page_header.rb
class Components::PageHeader < Components::Base
  def view_template(&)
    div(class: "page-header", &)
  end

  def title_bar(&)
    render TitleBar.new(&)
  end

  def description(&)
    div(class: "description", &)
  end
end
```

Usage in a view:

```ruby
render Components::PageHeader.new do |header|
  header.title_bar { |title_bar|
    title_bar.leading_action { # back button }
    title_bar.title { "My Page" }
    title_bar.actions { # action buttons }
  }
  header.description { "Optional subtitle" }
end
```

### Short-form Component Calls

`include Components` on `Components::Base` enables short-form rendering:

```ruby
# Instead of:
render Components::WebAwesome::WaIcon.new(name: "plus")

# Write:
WaIcon(name: "plus")
```

This works because `Components` is a module, and `include Components` makes nested constants available as methods via `method_missing` on Phlex components.

### Helper Includes

Each view includes only the specific `Phlex::Rails::Helpers::*` modules it needs:

```ruby
class Views::Labels::Show < Views::Base
  include Phlex::Rails::Helpers::Routes
  include Phlex::Rails::Helpers::ContentFor
  include Phlex::Rails::Helpers::DOMID
  include Phlex::Rails::Helpers::ButtonTo
  include Phlex::Rails::Helpers::LinkTo
  include Phlex::Rails::Helpers::TurboFrameTag
  # ...
end
```

Common helpers and their purpose:
- `Routes` — path/url helpers (usually on `Components::Base`)
- `ContentFor` — `content_for` / `content_for?` / `yield :name`
- `DOMID` — `dom_id(record)`
- `ButtonTo` — `button_to`
- `LinkTo` — `link_to`
- `TurboFrameTag` — `turbo_frame_tag`
- `TurboStreamFrom` — `turbo_stream_from`
- `FormTag` — `form_tag`
- `Request` — `request` object access
- `Notice` — flash notice
- `Debug` — `debug` helper
- `ImageTag` — `image_tag`
- `Sanitize` / `StripTags` — HTML sanitization
- `ClassNames` — `class_names` helper

### Content Areas

The layout yields named content areas that views populate via `content_for`:

```ruby
# In a view:
content_for :title, "Labels"

content_for :main_header do
  render Components::PageHeader.new do |header|
    header.title_bar { |title_bar|
      title_bar.title { "Labels" }
    }
  end
end

content_for :head do
  tag.script(type: "application/ld+json") { raw(safe(JSON.generate(schema))) }
end

content_for :floating_action do
  render Components::FloatingActionMenu.new
end
```

Standard content areas: `:title`, `:main_header`, `:floating_action`, `:head`.

### Custom Element Wrappers

Use `register_element` to wrap web component custom elements. This is the pattern for any custom element library (Web Awesome, Shoelace, etc.):

```ruby
module Components
  module MyLibrary
    class MyButton < Phlex::HTML
      register_element :my_button

      def initialize(variant: "neutral", size: "medium", disabled: false, href: nil, **attributes)
        @attributes = attributes.with_defaults(
          variant: variant,
          size: size,
          disabled: disabled,
          href: href
        )
      end

      def view_template(&)
        my_button(**@attributes, &)
      end
    end
  end
end
```

**Key points:**
- `register_element :element_name` creates a method matching the custom element tag
- Underscores become hyphens in the rendered HTML (`my_button` → `<my-button>`)
- All attributes collected in `initialize`, forwarded via `**@attributes`
- `attributes.with_defaults()` merges caller-provided overrides
- Content block captured with `&` and forwarded

For Web Awesome specifically, many elements have extensive attribute lists. The [`phlex_custom_element_generator`](https://github.com/konnorrogers/phlex_custom_element_generator) gem can auto-generate these wrappers from custom element manifests.

### Register Helpers

For Rails helpers that output HTML or return values:

```ruby
# In a component or layout:
register_output_helper :vite_client_tag       # helper returns HTML to render
register_output_helper :vite_javascript_tag
register_output_helper :wa_form_with          # custom form helper
register_output_helper :column_chart          # chartkick

register_value_helper :alert                  # helper returns a value (not HTML)
```

### Controller Rendering

Controllers render Phlex views directly, passing data via `new`:

```ruby
class LabelsController < ApplicationController
  def index
    @labels = Label.all
    @pagy, @labels = pagy(@labels)

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

### Fragment Caching

Views can cache content blocks:

```ruby
def view_template
  cache("labels/#{@label.id}/card") {
    # expensive rendering
  }
end

# cache_store defined in Views::Base:
def cache_store
  Rails.cache
end
```

### Multiple Layouts

Use separate layout classes for different sections (app shell vs. marketing):

```ruby
class Components::Layout < Components::Base
  include Phlex::Rails::Layout
  # ... app shell layout
end

class Components::MarketingLayout < Components::Base
  include Phlex::Rails::Layout
  # ... marketing/landing page layout
end
```

Set in controllers: `layout -> { Components::Layout }` or per-action.

### ERB Partials for Forms

Phlex views render ERB form partials seamlessly:

```ruby
# In a Phlex view:
section {
  render partial("form", label: @label)
}

# The ERB partial at app/views/labels/_form.html.erb:
# Uses standard form_with / form_for with Rails form builder
```

This is the pragmatic default — Rails form builders work well in ERB. Phlex form helpers exist for simple cases but ERB partials are preferred for complex forms.

### Grid / List Layout Components

```ruby
# app/components/grid.rb
class Components::Grid < Components::Base
  def initialize(id: nil, gap: "m", klass: nil)
    @id = id
    @gap = gap
    @class = klass
  end

  def view_template(&)
    div(id: @id, class: "grid gap-#{@gap} #{@class}", &)
  end
end

# app/components/grid_cell.rb
class Components::GridCell < Components::Base
  def initialize(id:, **options)
    @id = id
    @options = options
  end

  def view_template(&)
    div(id: @id, &)
  end
end
```

---

## Anti-patterns

- **God components** — a component with too many responsibilities; split into smaller, focused components
- **Business logic in views/components** — belongs in models, not rendering code
- **Over-including helpers on Base** — include `Phlex::Rails::Helpers::*` per-view, not on `Components::Base`
- **Reimplementing form builders in Phlex** — use ERB partials for forms when Rails form builders work fine
- **Monolithic views** — extract repeated UI into components; a view should compose components, not contain raw HTML for everything
- **Passing request context into components** — components receive data, not request objects; views can access `request` when needed

---

## Preferred Stack

- `phlex-rails` 2.x
- `vite_rails` (or importmap/esbuild)
- Pagy for pagination
- Custom element library of choice + [`phlex_custom_element_generator`](https://github.com/konnorrogers/phlex_custom_element_generator)
- Custom `PhlexControllerGenerator` for scaffold override (see Scaffolding section)
- `turbo-rails` + `stimulus-rails`
- Feature flags via Flipper (inline in views: `if Flipper.enabled?(:feature)`)

---

## Heuristics

- **Component vs View:** reusable UI → component; page-level (one per controller action) → view
- **When to use ERB partial:** forms with Rails form builder, third-party integrations, complex form logic
- **Naming:** `Components::PageHeader`, `Components::TitleBar`, `Views::Labels::Index`, `Views::Labels::Show`
- **One view per controller action;** views receive data via `initialize`
- **Prefer composition over inheritance** for components — use slot methods, not deep class hierarchies
- **Data flow:** controller → view (via `initialize`) → components (via `render Component.new(data)`)

---

## Frontend

### Stimulus

Data attributes as hash arguments:

```ruby
div(data: {
  controller: "faceted-search",
  action: "input->faceted-search#perform:prevent",
  faceted_search_url_value: labels_path
}) {
  # ...
}
```

### Turbo Frames

`turbo_frame_tag` with lazy loading:

```ruby
include Phlex::Rails::Helpers::TurboFrameTag

turbo_frame_tag(album, src: album_path(album), loading: :lazy) {
  # placeholder while loading
  render Components::Spinner.new
}
```

Turbo frame with Stimulus integration:

```ruby
turbo_frame_tag("chart",
                src: chart_path(@record, chart_type: :bar),
                loading: :lazy,
                data: {
                  controller: "frame-spinner",
                  action: "turbo:frame-load->chart-type-toggle#onFrameLoad"
                }) {
  template(data: { frame_spinner_target: "placeholder" }) {
    div(class: "loading-container") {
      render Components::Spinner.new
    }
  }
}
```

### Turbo Streams

```ruby
include Phlex::Rails::Helpers::TurboStreamFrom

turbo_stream_from([@budget, :items])
```

### Turbo Morphing

Set in layout `<head>`:

```ruby
meta name: "turbo-refresh-method", content: "morph"
meta name: "turbo-refresh-scroll", content: "preserve"
```

### Feature Flags

Inline conditional rendering:

```ruby
if Flipper.enabled?(:announcement)
  render Components::Callout.new(variant: "brand") {
    "New feature announcement here."
  }
end
```

### Pagy

Include `Pagy::Frontend` in views that paginate:

```ruby
class Views::Labels::Index < Views::Base
  include Pagy::Frontend
  include PagyHelper  # custom helper for styled pagination

  def initialize(labels, pagy)
    @labels = labels
    @pagy = pagy
  end

  def view_template
    # ... render labels ...

    # Render pagination
    raw safe(pagy_nav(@pagy))
  end
end
```

Controller passes Pagy metadata:

```ruby
def index
  @pagy, @labels = pagy(Label.all)
  render Views::Labels::Index.new(@labels, @pagy)
end
```

---

## Examples

### 1. Base + Views::Base Hierarchy

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

  def cache_store
    Rails.cache
  end
end
```

### 2. Component with Slot Methods (TitleBar)

```ruby
class Components::TitleBar < Components::Base
  def view_template(&)
    div(class: "title-bar", &)
  end

  def leading_action(&)
    div(class: "leading-action", &)
  end

  def title(&)
    h1(&)
  end

  def trailing_visual(&)
    div(&)
  end

  def actions(&)
    div(class: "actions", &)
  end
end
```

### 3. View with content_for and Controller Rendering

```ruby
# app/views/labels/show.rb
class Views::Labels::Show < Views::Base
  include Phlex::Rails::Helpers::Routes
  include Phlex::Rails::Helpers::ContentFor
  include Phlex::Rails::Helpers::DOMID
  include Phlex::Rails::Helpers::ButtonTo
  include Phlex::Rails::Helpers::LinkTo
  include Phlex::Rails::Helpers::TurboFrameTag

  def initialize(label)
    @label = label
  end

  def view_template
    content_for :title, @label.name

    content_for :main_header do
      render Components::PageHeader.new do |header|
        header.title_bar { |title_bar|
          title_bar.leading_action {
            link_to(labels_path) { "Back" }
          }

          title_bar.title { @label.name }

          title_bar.actions {
            button_to @label, method: :delete, data: { turbo_confirm: "Are you sure?" } do
              "Delete"
            end
          }
        }
      end
    end

    section {
      p { @label.description }

      h4 { "Recent albums:" }
      @label.albums.limit(6).each do |album|
        turbo_frame_tag(album, src: album_path(album), loading: :lazy) {
          render Components::Spinner.new
        }
      end
    }
  end
end

# In the controller:
class LabelsController < ApplicationController
  def show
    @label = Label.find(params[:id])
    render Views::Labels::Show.new(@label)
  end
end
```

### 4. Custom Element Wrapper Pattern

```ruby
# app/components/my_library/my_button.rb
module Components
  module MyLibrary
    class MyButton < Phlex::HTML
      register_element :my_button

      def initialize(
        variant: "neutral",
        size: "medium",
        disabled: false,
        href: nil,
        type: "button",
        **attributes
      )
        @attributes = attributes.with_defaults(
          variant: variant,
          size: size,
          disabled: disabled,
          href: href,
          type: type
        )
      end

      def view_template(&)
        my_button(**@attributes, &)
      end
    end
  end
end

# Renders: <my-button variant="brand" size="small">Click me</my-button>
render Components::MyLibrary::MyButton.new(variant: :brand, size: :small) { "Click me" }
```

For Web Awesome specifically, the wrapper follows this identical pattern with `wa_button`, `wa_icon`, `wa_card`, etc. — each mapping to `<wa-button>`, `<wa-icon>`, `<wa-card>`.

### 5. Generator Code + Index Template

See [Scaffolding / Generator](#scaffolding--generator) section above for the complete generator and all five templates.

### 6. Turbo Frame Lazy Loading

```ruby
class Views::Albums::Show < Views::Base
  include Phlex::Rails::Helpers::TurboFrameTag

  def initialize(album)
    @album = album
  end

  def view_template
    turbo_frame_tag(@album) {
      # Full album content rendered here when loaded directly
      div { @album.title }
    }
  end
end

# In another view, lazy-load the album frame:
turbo_frame_tag(album, src: album_path(album), loading: :lazy) {
  # Placeholder shown until frame loads
  render Components::Spinner.new
}
```

### 7. ERB Form Partial Rendered from Phlex View

```ruby
# app/views/labels/new.rb
class Views::Labels::New < Views::Base
  include Phlex::Rails::Helpers::Routes
  include Phlex::Rails::Helpers::ContentFor

  def initialize(label)
    @label = label
  end

  def view_template
    content_for :title, "New label"

    content_for :main_header do
      render Components::PageHeader.new do |header|
        header.title_bar { |title_bar|
          title_bar.title { "New label" }
        }
      end
    end

    section {
      render partial("form", label: @label)
    }
  end
end
```

```erb
<%# app/views/labels/_form.html.erb %>
<%# locals: (label:) %>
<%= form_with(model: label, class: "form-stack") do |form| %>
  <% if label.errors.any? %>
    <div class="error-callout">
      <strong><%= pluralize(label.errors.count, "error") %></strong>
      <ul>
        <% label.errors.each do |error| %>
          <li><%= error.full_message %></li>
        <% end %>
      </ul>
    </div>
  <% end %>

  <%= form.text_field :name %>
  <%= form.url_field :url %>
  <%= form.text_area :description %>
  <%= form.submit %>
<% end %>
```

---

## Testing

See [shared/testing.md](shared/testing.md) for the testing approach. Phlex components and views are tested as Ruby objects — instantiate them and assert on the rendered HTML output.
