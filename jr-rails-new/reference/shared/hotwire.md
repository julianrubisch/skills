# Hotwire Reference

Concise overview of the Hotwire stack (Turbo + Stimulus) for Rails. The
`hwc-*` skills provide deeper coverage per domain — this file is the
connecting reference.

> **Skills:** `hwc-stimulus-fundamentals`, `hwc-navigation-content`,
> `hwc-forms-validation`, `hwc-ux-feedback`, `hwc-realtime-streaming`,
> `hwc-media-content`

## Stimulus Controllers

### Basic Structure

```javascript
// app/javascript/controllers/dropdown_controller.js
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static targets = ["menu"]
  static classes = ["open"]
  static values = { open: { type: Boolean, default: false } }

  toggle() {
    this.openValue = !this.openValue
  }

  openValueChanged() {
    this.menuTarget.classList.toggle(this.openClass, this.openValue)
  }

  closeOnClickOutside(event) {
    if (!this.element.contains(event.target)) {
      this.openValue = false
    }
  }
}
```

### Controller Communication

Prefer **outlets** for direct references, **custom events** for loose coupling:

```javascript
// Outlets — direct controller-to-controller
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static outlets = ["search-results"]

  filter() {
    if (this.hasSearchResultsOutlet) {
      this.searchResultsOutlet.updateResults(this.query)
    }
  }
}

// Custom events — decoupled
this.dispatch("filter", { detail: { query }, prefix: "search" })
```

> Deep dive: `hwc-stimulus-fundamentals` — lifecycle, values, targets,
> outlets, action parameters, keyboard events.

## Turbo Frames

### Partial Page Updates

```erb
<turbo-frame id="posts">
  <% @posts.each do |post| %>
    <turbo-frame id="<%= dom_id(post) %>">
      <%= render post %>
    </turbo-frame>
  <% end %>
</turbo-frame>
```

### Lazy Loading

```erb
<%= turbo_frame_tag "comments",
      src: post_comments_path(@post),
      loading: :lazy do %>
  <p>Loading comments...</p>
<% end %>
```

### Breaking Out of Frames

```erb
<%# Navigate the whole page, not the frame %>
<%= link_to "View", post_path(post), data: { turbo_frame: "_top" } %>
```

> Deep dive: `hwc-navigation-content` — pagination, tabs, lazy loading,
> filtering, cache lifecycle, scroll restoration.

## Turbo Streams

### Stream Actions

Seven built-in actions plus `refresh`:

| Action | Effect |
|--------|--------|
| `append` / `prepend` | Add to container |
| `replace` | Replace entire element |
| `update` | Replace inner HTML only |
| `remove` | Remove element |
| `before` / `after` | Insert adjacent |
| `refresh` | Trigger page refresh (uses morphing if configured) |

```erb
<%# app/views/posts/create.turbo_stream.erb %>
<%= turbo_stream.prepend "posts", @post %>
<%= turbo_stream.update "posts-count", @posts.count %>

<%# Replace with morphing for smoother updates %>
<%= turbo_stream.replace @post, method: :morph %>

<%# Trigger full page refresh (morphing-aware) %>
<%= turbo_stream.refresh %>
```

### Controller Responses

```ruby
def update
  @post = Post.find(params[:id])

  if @post.update(post_params)
    respond_to do |format|
      format.html { redirect_to @post }
      format.turbo_stream
    end
  else
    render :edit, status: :unprocessable_entity
  end
end
```

### CSS Selector Targeting

```erb
<%# Target multiple elements by CSS selector %>
<%= turbo_stream.remove_all ".notification" %>
<%= turbo_stream.update_all ".counter", "0" %>
```

> Deep dive: `hwc-forms-validation` — submission lifecycle, inline editing,
> validation errors, modal forms.

## Broadcasting (Real-Time)

### Model Callbacks

```ruby
class Message < ApplicationRecord
  belongs_to :room

  # Automatic broadcasts on create/update/destroy
  broadcasts_to :room

  # With custom configuration
  broadcasts_to ->(message) { [message.room, :messages] },
    inserts_by: :prepend,
    target: "room_messages"
end
```

### Refresh Broadcasting (Turbo 8+)

Instead of surgically updating individual elements, broadcast a page refresh
that uses **morphing** to reconcile changes smoothly:

```ruby
class Article < ApplicationRecord
  # Declarative — broadcasts refresh on create/update/destroy
  broadcasts_refreshes_to :category

  # Or manual in callbacks
  after_update_commit -> { broadcast_refresh_later_to self }
end
```

Client-side, enable morphing in the `<head>`:

```erb
<%= turbo_refreshes_with method: :morph, scroll: :preserve %>
```

**When to use refresh vs surgical streams:**
- **Refresh** — page has many interdependent elements, complex state,
  or you want zero template duplication (server renders once)
- **Surgical** (`append`/`replace`/`remove`) — targeted updates where
  you control exactly what changes

### Async Broadcasting

Use `_later` variants in callbacks to avoid blocking the request:

```ruby
after_create_commit -> { broadcast_append_later_to room, :messages }
```

### Subscribing on the Client

```erb
<%= turbo_stream_from @room %>
```

### Suppressing Broadcasts

```ruby
Message.suppressing_turbo_broadcasts do
  Message.create!(content: "This won't broadcast")
end
```

> Deep dive: `hwc-realtime-streaming` — WebSocket/SSE, custom stream
> actions, cross-tab sync, stream action orchestration.

## Custom Stream Actions

```javascript
// app/javascript/application.js
import { Turbo } from "@hotwired/turbo-rails"

Turbo.StreamActions.notification = function() {
  const message = this.getAttribute("message")
  // Show notification via your UI system
}
```

```erb
<%= turbo_stream_action_tag "notification", message: "Post created!" %>
```

## Performance Patterns

### Turbo Permanent Elements

Elements that persist across navigations (flash containers, audio players):

```erb
<div id="flash-messages" data-turbo-permanent>
  <%= render "shared/flash" %>
</div>
```

### Cache Control

```erb
<meta name="turbo-cache-control" content="no-preview">
<meta name="turbo-cache-control" content="no-cache">
```

### Auto-Submit with Debounce

```javascript
import { Controller } from "@hotwired/stimulus"

export default class extends Controller {
  static values = { delay: { type: Number, default: 300 } }

  connect() {
    this.timeout = null
  }

  submit() {
    clearTimeout(this.timeout)
    this.timeout = setTimeout(() => {
      this.element.requestSubmit()
    }, this.delayValue)
  }
}
```

> Deep dive: `hwc-ux-feedback` — loading states, busy indicators,
> optimistic UI, view transitions, morph reconciliation.

## See Also

- `coding-classic.md` — Hotwire conventions within Rails coding style
- `shared/components.md` — Phlex components with Stimulus data attributes
- `shared/jobs.md` — background jobs for async broadcast sources
