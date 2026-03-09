# Notifications

## Summary

Notifications deliver messages through multiple channels (email, SMS, push, in-app). The Noticed gem provides a layer of abstraction over delivery mechanisms, making notifications a first-class domain concept with database-backed persistence.

## Layer Placement

```
┌─────────────────────────────────────────┐
│ Application Layer                       │
│  └─ Notifier classes (what to send)     │
├─────────────────────────────────────────┤
│ Infrastructure Layer                    │
│  └─ Mailers, SMS adapters (how to send) │
│  └─ Notification records (persistence)  │
└─────────────────────────────────────────┘
```

## Key Principles

- **Delivery abstraction** — separate what from how
- **Channel-agnostic** — same notification, multiple delivery methods
- **Database-backed** — persistent notification records for in-app feeds
- **Testable** — verify notifications without infrastructure
- **Configurable** — enable/disable channels per notification type

## Implementation with Noticed

### Setup

```bash
bundle add "noticed"
rails noticed:install:migrations
rails db:migrate
```

```ruby
# app/models/user.rb
class User < ApplicationRecord
  has_many :notifications, as: :recipient, dependent: :delete_all
end
```

### Basic Notifier

```ruby
# app/notifiers/post_published_notifier.rb
class PostPublishedNotifier < Noticed::Event
  deliver_by :email do |config|
    config.mailer = "PostsMailer"
    config.method = :published
  end

  required_params :post

  notification_methods do
    def message
      t(".message", title: params[:post].title)
    end

    def url
      post_path(params[:post])
    end
  end
end
```

```ruby
# app/mailers/posts_mailer.rb
class PostsMailer < ApplicationMailer
  def published
    @post = params[:post]
    @notification = params[:notification]

    mail(
      to: params[:recipient].email,
      subject: "Your post '#{@post.title}' is now live!"
    )
  end
end
```

### Multi-Channel Delivery

```ruby
class PostPublishedNotifier < Noticed::Event
  deliver_by :email do |config|
    config.mailer = "PostsMailer"
    config.method = :published
  end

  deliver_by :action_cable do |config|
    config.channel = "NotificationsChannel"
    config.stream = -> { "notifications_#{recipient.id}" }
  end

  bulk_deliver_by :slack do |config|
    config.url = -> { Rails.application.credentials.slack_webhook_url }
    config.json = -> {
      { text: "New post published: #{params[:post].title}" }
    }
  end

  required_params :post
end
```

### Triggering Notifications

```ruby
# In controller or domain object
class PostsController < ApplicationController
  def publish
    @post = Post.find(params[:id])
    authorize @post, :publish?

    @post.publish!
    PostPublishedNotifier.with(post: @post).deliver(@post.subscribers)
  end
end

# Or from a state machine after_commit callback
class Post < ApplicationRecord
  include AASM

  aasm column: :status do
    event :publish, after_commit: :notify_published do
      transitions from: :approved, to: :published
    end
  end

  private

  def notify_published
    PostPublishedNotifier.with(post: self).deliver(subscribers)
  end
end

# Or via EventReporter (Rails 8.1+) — fully decoupled
class Post < ApplicationRecord
  after_commit on: :update do
    Rails.event.notify("post.published", post_id: id) if saved_change_to_status? && published?
  end
end

# Subscriber handles notification delivery — model doesn't know about Noticed
class PostEventSubscriber
  def emit(event)
    case event[:name]
    when "post.published"
      post = Post.find(event[:payload][:post_id])
      PostPublishedNotifier.with(post: post).deliver(post.subscribers)
    end
  end
end

# config/initializers/event_subscribers.rb
Rails.event.subscribe(PostEventSubscriber.new) { |e| e[:name].start_with?("post.") }
```

### Conditional Delivery

```ruby
class PostPublishedNotifier < Noticed::Event
  deliver_by :email do |config|
    config.mailer = "PostsMailer"
    config.method = :published
    config.if = -> { recipient.notification_settings&.email_enabled? }
  end

  deliver_by :action_cable do |config|
    config.channel = "NotificationsChannel"
    config.if = -> { recipient.notification_settings&.push_enabled? }
  end

  # Skip expensive checks before enqueuing
  deliver_by :ios do |config|
    config.before_enqueue = -> { throw(:abort) unless recipient.registered_ios? }
  end
end
```

### In-App Notification Feed

```ruby
# Controller
class NotificationsController < ApplicationController
  def index
    @notifications = current_user.notifications
                                 .includes(:event)
                                 .order(created_at: :desc)
  end

  def mark_read
    current_user.notifications.where(id: params[:ids]).update_all(read_at: Time.current)
    head :ok
  end
end

# View (ERB)
<% @notifications.each do |notification| %>
  <div class="notification <%= 'unread' if notification.read_at.nil? %>">
    <p><%= notification.message %></p>
    <%= link_to "View", notification.url %>
  </div>
<% end %>
```

### Batch / Digest Notifications

```ruby
class DailyDigestNotifier < Noticed::Event
  deliver_by :email do |config|
    config.mailer = "DigestMailer"
    config.method = :daily
  end

  required_params :posts
end

# Scheduled job
class DailyDigestJob < ApplicationJob
  def perform
    User.where(digest_enabled: true).find_each do |user|
      posts = user.unread_posts.where(created_at: 1.day.ago..)
      next if posts.empty?

      DailyDigestNotifier.with(posts: posts).deliver(user)
    end
  end
end
```

## Without Noticed

For apps that don't need database-backed notifications or multi-channel
delivery, a plain service object works fine:

```ruby
class NotifyPostPublished
  def initialize(post)
    @post = post
  end

  def call
    send_email
    send_push if post.author.push_enabled?
    send_slack if post.featured?
  end

  private

  attr_reader :post

  def send_email
    PostsMailer.published(post).deliver_later
  end

  def send_push
    PushService.deliver(
      user: post.author,
      title: "Post Published",
      body: "Your post '#{post.title}' is now live!"
    )
  end

  def send_slack
    SlackService.post(
      channel: "#content",
      text: "New featured post: #{post.title}"
    )
  end
end
```

## Anti-Patterns

### Notifications in Models

```ruby
# BAD: Domain layer sending notifications
class Post < ApplicationRecord
  after_update :notify_if_published

  private

  def notify_if_published
    return unless saved_change_to_published_at?
    PostsMailer.published(self).deliver_later
  end
end

# GOOD: Notifications from state machine or controller
PostPublishedNotifier.with(post: post).deliver(post.subscribers)
```

### Mailer Knows Too Much

```ruby
# BAD: Mailer with business logic
class PostsMailer < ApplicationMailer
  def published(post)
    @post = post
    notify_editors(post) if post.featured?
    mail(to: post.author.email, subject: subject_for(post))
  end
end

# GOOD: Mailer only formats and sends
class PostsMailer < ApplicationMailer
  def published
    @post = params[:post]
    mail(to: params[:recipient].email, subject: "Post published!")
  end
end

# Channel routing in notifier
class PostPublishedNotifier < Noticed::Event
  deliver_by :email do |config|
    config.mailer = "PostsMailer"
  end

  deliver_by :email do |config|
    config.mailer = "EditorMailer"
    config.if = -> { params[:post].featured? }
  end
end
```

### Inline Notification Logic

```ruby
# BAD: Scattered notification logic
class PostsController < ApplicationController
  def publish
    @post.publish!
    PostsMailer.published(@post).deliver_later
    SlackService.post(channel: "#content", text: "...")
    PushService.deliver(user: @post.author, ...)
  end
end

# GOOD: Encapsulated in notifier
class PostsController < ApplicationController
  def publish
    @post.publish!
    PostPublishedNotifier.with(post: @post).deliver(@post.subscribers)
  end
end
```

## Triggering from State Machines

When notifications are tied to state transitions, trigger from `after_commit`
callbacks on AASM events — keeps models free of notification logic while
centralizing state-related side effects.

> See `shared/state_machines.md § Triggering Deliveries` for the pattern.

## Testing

```ruby
class PostPublishedNotifierTest < ActiveSupport::TestCase
  test "delivers email to subscribers" do
    post = posts(:approved_post)

    assert_enqueued_email_with PostsMailer, :published do
      PostPublishedNotifier.with(post: post).deliver(users(:subscriber))
    end
  end

  test "skips email when user has email disabled" do
    post = posts(:approved_post)
    user = users(:no_email_user)

    assert_no_enqueued_emails do
      PostPublishedNotifier.with(post: post).deliver(user)
    end
  end

  test "creates in-app notification record" do
    post = posts(:approved_post)
    user = users(:subscriber)

    assert_difference "user.notifications.count", 1 do
      PostPublishedNotifier.with(post: post).deliver(user)
    end
  end
end
```

## Related Gems

| Gem | Purpose |
|-----|---------|
| [noticed](https://github.com/excid3/noticed) | Multi-channel notifications with database persistence |
