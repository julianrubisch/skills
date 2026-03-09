# Performance Review Reference

## Audit Framework

A performance review has four sections — work through them in order:

1. **Frontend** — everything after HTML leaves the server (Core Web Vitals / PageSpeed, images, lazy loading, preloading) — defer to hwc-* skills for browser/JS specifics
2. **Database** — I/O between app and DB (N+1, indexes, query count, eager loading)
3. **Ruby/Memory** — timing and memory benchmarks of application code
4. **Environment** — app server config, cache stores, load testing

### Tooling

Add to bundle for profiling (remove after audit):

| Concern | Gem |
|---------|-----|
| N+1 detection | `prosopite` |
| Request profiling | `rack-mini-profiler` |
| Memory profiling | `memory_profiler` |
| CPU profiling | `stackprof`, `ruby-prof` |
| Require-time memory | `derailed_benchmarks` |
| Process memory | `get_process_mem` |

Global installs:
- `fasterer` — Ruby benchmark linting
- `bumbler` — application load time by require (`bumbler -t 50`)

Load testing: `oha`, `wrk`, or `k6` (cloud). Example: `oha -q 2 -z 20s --disable-keepalive --latency-correction`

---

## Patterns

### HTTP Caching with Conditional GET

Three separate `etag` concerns — keep them distinct:

```ruby
# 1. Global version buster — bump "v1" to invalidate all ETags app-wide
class ApplicationController < ActionController::Base
  etag { "v1" }
end

# 2. User-scope guard — prevents serving cached content across accounts
class ApplicationController < ActionController::Base
  etag { current_user&.id }
end

# 3. Per-action ETags — record-based (show) and collection-based (index)
class CardsController < ApplicationController
  def show
    @card = Card.find(params[:id])
    fresh_when etag: [@card, Current.user.timezone]
    # Timezone must be in ETag: times render server-side in user's timezone
  end

  def index
    @cards = @board.cards.preloaded
    fresh_when etag: [@cards, @board.updated_at]
  end
end
```

Use `expires_in` for time-based expiry on public, shared content (no user context):

```ruby
def index
  @articles = Article.published
  expires_in 15.minutes, public: true
end
```

### Fragment and Russian Doll Caching

Cache partials at the fragment level. Nest cache blocks for automatic
invalidation propagated via `touch: true`.

```erb
<%# Russian doll: board cache invalidates when any card changes %>
<% cache @board do %>
  <% @board.cards.each do |card| %>
    <% cache card do %>
      <%= render card %>
    <% end %>
  <% end %>
<% end %>
```

```ruby
# touch: true propagates updated_at up the association chain
class Comment < ApplicationRecord
  belongs_to :card,  touch: true
end

class Card < ApplicationRecord
  belongs_to :board, touch: true
end
# Comment update → card.updated_at changes → board.updated_at changes → cache busted
```

### `.size` vs `.count` on Loaded Relations

If a relation has already been loaded, count it in memory. `.count` fires a
new SQL query regardless.

```ruby
@posts = Post.all
@posts.count  # BAD — new COUNT query
@posts.size   # GOOD — counts in memory if already loaded
```

Only applies when the relation is actually used downstream. If you never
iterate `@posts`, use `Post.count` directly.

### Turbo Frame Lazy Loading for Expensive Sub-views

Associations shown on a show page (e.g. a user's posts, a project's tasks)
don't need to be in the initial response. Wrap them in a lazy Turbo Frame.

```erb
<%# Renders a spinner, then fetches content separately %>
<%= turbo_frame_tag "user_posts", src: user_posts_path(@user), loading: :lazy %>
```

This defers the DB query and rendering to a separate request, improving
initial response time and perceived performance.

### Preloading Hero Images

Add a `<link rel="preload">` for the most prominent image on a show/edit page
to improve Largest Contentful Paint.

```erb
<%# In layout or view %>
<%= preload_link_tag url_for(@resource.cover_image), as: :image %>
```

### ViewComponent Collection Rendering

Use `render ComponentName.with_collection(items)` instead of iterating with
`each`. ViewComponent's collection rendering is significantly faster.

```ruby
# GOOD
<%= render Avo::Index::GridItemComponent.with_collection(@resources) %>

# BAD — renders each component separately
<% @resources.each do |resource| %>
  <%= render Avo::Index::GridItemComponent.new(resource: resource) %>
<% end %>
```

### Eager Loading Associations

Declare all associations touched in a view up front. Includes ActiveStorage —
its `Attachment` and `Blob` models are associations like any other.

```ruby
# GOOD
def index
  @posts = Post.includes(:author, :tags, image_attachment: :blob)
end
```

### Async Queries for Independent DB Calls (Rails 7.0.1+)

When a controller action makes two independent queries (e.g. results + count
for pagination), run them concurrently with `load_async`.

```ruby
# GOOD — count and results run in parallel
def index
  @posts = Post.published.order(:created_at).load_async
  @count = Post.published.async_count
end
```

### Full-text Search via Materialized View + GIN Index

For cross-table full-text search, a materialized view with GIN index outperforms
JOIN-based `pg_search` by ~300x on large datasets. Manage with the `scenic` gem.

```sql
-- db/views/searchable_posts_v01.sql
SELECT posts.id,
       to_tsvector('english',
         coalesce(posts.title, '') || ' ' ||
         coalesce(authors.name, '') || ' ' ||
         coalesce(string_agg(tags.name, ' '), '')
       ) AS search_vector
FROM posts
JOIN authors ON authors.id = posts.author_id
LEFT JOIN taggings ON taggings.post_id = posts.id
LEFT JOIN tags ON tags.id = taggings.tag_id
GROUP BY posts.id, posts.title, authors.name
```

```ruby
# Migration
add_index :searchable_posts, :search_vector, using: :gin
```

Sync the view via a background job triggered by model callbacks on write.

### Database Index Diagnostics

Two SQL queries worth running during any performance review:

```sql
-- 1. Find tables likely missing indexes (skip tables < 80KB — seq scan wins there)
SELECT relname,
       seq_scan - idx_scan AS too_much_seq,
       CASE WHEN seq_scan - coalesce(idx_scan, 0) > 0
            THEN 'Missing Index' ELSE 'OK' END AS status,
       pg_relation_size(relname::regclass) AS rel_size
FROM pg_stat_all_tables
WHERE schemaname = 'public'
  AND pg_relation_size(relname::regclass) > 80000
ORDER BY too_much_seq DESC;

-- 2. Measure index efficiency (cache hit ratio — sort ASC to find worst offenders)
SELECT relname AS table_name,
       (idx_blks_hit * 1.0 / (idx_blks_hit + idx_blks_read)) AS index_efficiency
FROM pg_statio_user_tables
WHERE idx_blks_read + idx_blks_hit > 0
ORDER BY index_efficiency ASC
LIMIT 10;
```

## Anti-patterns

### N+1 Queries

**Problem:** Associations loaded inside a loop — one query per record.
Especially common with ActiveStorage, which has two levels of association
(`Attachment` → `Blob`).

```ruby
# BAD — N+1 on attachment, then N+1 on blob
def index
  @users = User.all
end
# In view: user.avatar.url → hits DB for every user, twice

# GOOD
def index
  @users = User.includes(avatar_attachment: :blob)
end
```

ActiveStorage N+1 has two levels — the `Attachment` record and the `Blob` record.
ActiveStorage provides scoped helpers for bulk-loading:

```ruby
# For specific attachments
User.with_attached_avatar          # eager loads avatar + blob
Post.with_attached_images          # eager loads images + blobs

# In an index action
@users = User.all.with_attached_avatar.with_attached_cover
```

Detection tools: `prosopite` middleware (preferred), or `strict_loading!` on
the relation to surface unloaded associations with exceptions during development.

**Signal:** Bullet gem warnings, or `pg_stat_statements` showing the same
query repeated N times with different ID values.

### No HTTP Caching on Expensive Public Endpoints

**Problem:** Every request for the same public content hits the server and
database, even when the content hasn't changed.

**Signal:** High-traffic index or show actions with no `fresh_when`,
`expires_in`, or cache headers in the response.

**Fix:** `fresh_when(@record)` for record-based pages, `expires_in` for
time-stable public pages. Always scope ETags to `current_user&.id` to prevent
cross-user cache leaks.

### Sequential Scans on Large Tables

**Problem:** Foreign key columns or frequently filtered columns without indexes
cause full table scans as data grows.

**Signal:** `seq_scan - idx_scan` is positive and large in the diagnostic query
above. Common culprits: polymorphic `*_id` + `*_type` columns, `created_at`
on tables filtered by date range.

### Async Queries Without Connection Pool Awareness

**Problem:** `load_async` borrows an extra DB connection. With Puma + Solid Queue
on a small pool, this causes `ConnectionTimeoutError` under load.

**Fix:** Size the pool to account for async usage:
`pool = puma_threads + job_concurrency + async_headroom`

## Heuristics

- Run the two index diagnostic queries on any app before claiming it's "fast"
- Every `has_many` or `belongs_to` touched in a view is a potential N+1
- ActiveStorage counts as two associations: always `includes(x_attachment: :blob)`
- `load_async` is free performance when two queries are independent — use it
- Materialized views are a read/write tradeoff: only worth it when search is frequent and write latency is acceptable
- Sequential scan is fine on tables under ~80KB — don't index everything
- Timezone in the ETag is not optional when times render server-side
