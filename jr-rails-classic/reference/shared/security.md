# Security: Shared Reference

Prescriptive guide — how to write secure Rails code. Loaded by both coding
skills and review agents. Review-specific concerns (what to flag, severity)
live in review-security.md.

## SQL Injection

Use parameterized queries. Never interpolate user input into SQL strings.

```ruby
# BAD
where("name = '#{params[:name]}'")
where("name LIKE '%#{term}%'")
find_by_sql("SELECT * FROM users WHERE id = #{id}")
order("#{params[:sort]} #{params[:direction]}")

# GOOD
where("name = ?", params[:name])
where("name LIKE ?", "%#{term}%")
where(name: params[:name])
order(Arel.sql(safe_order_string))
```

## XSS Prevention

Escape user content before rendering. Never call `.html_safe` on user-supplied
strings directly.

```ruby
# GOOD — escape first, then mark safe
def formatted_content(text)
  simple_format(h(text)).html_safe
end
```

```erb
<%# BAD — raw output %>
<%= raw user_input %>
<%= user_input.html_safe %>
<%== user_input %>

<%# GOOD — auto-escaped or sanitized %>
<%= user_input %>
<%= sanitize(user_input) %>
```

```javascript
// BAD
element.innerHTML = userInput;

// GOOD
element.textContent = userInput;
```

## Strong Parameters

Always whitelist at the controller boundary. Never pass `params` directly
to model methods.

```ruby
# BAD
params.permit!
User.new(params[:user])
update_attributes(params)

# GOOD
def user_params
  params.require(:user).permit(:name, :email)
end

User.new(user_params)
```

## Authorization Scoping

Scope all record lookups to the current user or account. Never look up
a record by ID alone without verifying ownership.

```ruby
# BAD — any authenticated user can access any record
@document = Document.find(params[:id])

# GOOD — scoped to current account
@document = Current.account.documents.find(params[:id])

# GOOD — Pundit authorization
@document = Document.find(params[:id])
authorize @document
```

Use Pundit policies for non-trivial authorization logic (see `patterns.md §
Policy Objects`).

## SSRF Protection

Resolve DNS once and pin the IP to prevent TOCTOU attacks on URL fetching.

```ruby
def fetch_safely(url)
  uri = URI.parse(url)
  ip  = Resolv.getaddress(uri.host)

  raise "Private IP" if private_ip?(ip)

  Net::HTTP.start(uri.host, uri.port, ipaddr: ip) { |http| http.get(uri.path) }
end

def private_ip?(ip)
  ip.start_with?("127.", "10.", "192.168.") ||
    ip.match?(/^172\.(1[6-9]|2[0-9]|3[0-1])\./)
end
```

## Content Security Policy

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.default_src    :self
    policy.script_src     :self
    policy.style_src      :self, :unsafe_inline
    policy.base_uri       :none
    policy.form_action    :self
    policy.frame_ancestors :self
  end
end
```

## CSP Nonces

For apps using importmap or inline Stimulus controllers, use nonce-based CSP
instead of `unsafe-inline` to allow only Rails-generated script tags.

```ruby
# config/initializers/content_security_policy.rb
Rails.application.configure do
  config.content_security_policy do |policy|
    policy.script_src :self, :nonce
  end
  config.content_security_policy_nonce_generator = ->(request) { request.session.id.to_s }
  config.content_security_policy_nonce_directives = %w[script-src]
end
```

```erb
<%# Rails auto-injects nonce on script tags when using helpers %>
<%= javascript_importmap_tags nonce: true %>
```

## ActionText Sanitization

Restrict allowed tags explicitly — don't rely on ActionText defaults.

```ruby
# config/initializers/action_text.rb
Rails.application.config.after_initialize do
  ActionText::ContentHelper.allowed_tags = %w[
    strong em a ul ol li p br h1 h2 h3 h4 blockquote
  ]
end

# Rails 7.1+ — can also configure via sanitizer class
# config.action_text.sanitizer = Rails::HTML5::SafeListSanitizer.new
```

## Command Injection

```ruby
# BAD — shell interprets user input
system("convert #{params[:file]}")
`ls #{user_input}`
exec("command #{args}")
%x(command #{args})

# GOOD — array form bypasses shell interpretation
system("convert", params[:file])
```

## Path Traversal

```ruby
# BAD — user controls file path
send_file(params[:filename])
File.read(params[:path])
render file: params[:template]

# GOOD — restrict to basename, validate path
basename = File.basename(params[:filename])
safe_path = Rails.root.join("uploads", basename)
send_file(safe_path) if File.exist?(safe_path)
```

## Sensitive Data Exposure

```ruby
# BAD — logging or serializing sensitive data
Rails.logger.info("Password: #{params[:password]}")
Rails.logger.info(params.inspect)
render json: user  # may include password_digest, tokens

# GOOD — filter params, whitelist JSON
config.filter_parameters += [:password, :token, :secret]
render json: user.as_json(only: [:id, :name, :email])
```

## Encrypted Credentials

Use Rails encrypted credentials instead of scattered ENV vars. Credentials are
encrypted at rest, version-controlled, and structured.

```ruby
# Edit credentials
# bin/rails credentials:edit
# bin/rails credentials:edit --environment production

# Access in code
Rails.application.credentials.dig(:aws, :access_key_id)
Rails.application.credentials.secret_key_base

# BAD — unencrypted, scattered across .env files
ENV["AWS_ACCESS_KEY_ID"]
ENV["STRIPE_SECRET_KEY"]

# GOOD — encrypted, version-controlled, structured
Rails.application.credentials.aws[:access_key_id]
Rails.application.credentials.stripe[:secret_key]
```

Keep `config/master.key` (or per-environment keys) out of version control.
Share keys via secure channels, never commit them.

## Cryptography

```ruby
# BAD — weak or non-encryption
Digest::MD5.hexdigest(password)
Digest::SHA1.hexdigest(password)
Base64.encode64(secret)  # encoding, not encryption

# GOOD
BCrypt::Password.create(password)        # via has_secure_password
ActiveSupport::MessageEncryptor           # for symmetric encryption
```

## Session Security

```ruby
# GOOD — secure session configuration
Rails.application.config.session_store :cookie_store,
  key: '_app_session',
  secure: Rails.env.production?,
  httponly: true,
  same_site: :lax,
  expire_after: 30.minutes
```

## SameSite Cookies

Control when cookies are sent on cross-site requests. Prevents CSRF from
third-party origins without relying solely on CSRF tokens.

```ruby
# config/initializers/session_store.rb
Rails.application.config.session_store :cookie_store,
  key: '_app_session',
  same_site: :lax  # default — safe for most apps

# :strict — cookie never sent on cross-site requests (breaks OAuth callback flows)
# :lax    — cookie sent on top-level navigations only (GET from external link)
# :none   — cookie sent everywhere (requires secure: true, needed for iframe embeds)
```

Use `:lax` as default. Only use `:none` when the app is embedded in iframes
on third-party sites (e.g., Shopify apps). Always pair `:none` with `secure: true`.

## CSRF Protection

```ruby
# BAD — disabling CSRF entirely
class ApiController < ApplicationController
  skip_before_action :verify_authenticity_token
end

# GOOD — null session for stateless APIs, keep CSRF for browser requests
class ApiController < ApplicationController
  protect_from_forgery with: :null_session
end

# Always use Rails form helpers (include CSRF token automatically)
<%= form_with url: posts_path do |f| %>
```

## Rate Limiting

### Rails 8+ Built-in Rate Limiting

Rails 8 provides controller-level rate limiting out of the box. Backed by
`ActiveSupport::Cache` — no extra gems needed.

```ruby
class SessionsController < ApplicationController
  # 10 login attempts per 3 minutes, keyed by IP
  rate_limit to: 10, within: 3.minutes, only: :create

  # Custom key — rate limit per email to prevent credential stuffing
  rate_limit to: 5, within: 1.minute, only: :create,
    by: -> { params.dig(:session, :email)&.downcase }

  # Custom response when rate limited
  rate_limit to: 10, within: 3.minutes, only: :create,
    with: -> { redirect_to new_session_path, alert: "Too many attempts. Try later." }
end

class Api::BaseController < ApplicationController
  # API-wide rate limit per token
  rate_limit to: 100, within: 1.minute,
    by: -> { request.headers["Authorization"] }
end
```

**Prefer Rails 8 built-in** for controller-scoped throttling. It's simple,
requires no dependencies, and integrates with the existing cache store.

### Rack::Attack (pre-Rails 8 or middleware-level)

Use Rack::Attack when you need middleware-level throttling (before routing),
IP blocklists, or ban-on-repeat-offender patterns.

```ruby
# config/initializers/rack_attack.rb

# General throttle — 300 requests per 5 minutes per IP
Rack::Attack.throttle("req/ip", limit: 300, period: 5.minutes) do |req|
  req.ip
end

# Login throttle — 5 attempts per 20 seconds per IP
Rack::Attack.throttle("logins/ip", limit: 5, period: 20.seconds) do |req|
  req.ip if req.path == "/session" && req.post?
end

# Login throttle — per email (prevents credential stuffing)
Rack::Attack.throttle("logins/email", limit: 5, period: 1.minute) do |req|
  req.params.dig("session", "email")&.downcase if req.path == "/session" && req.post?
end

# Ban repeat offenders
Rack::Attack.blocklist("block bad IPs") do |req|
  Rack::Attack::Allow2Ban.filter(req.ip, maxretry: 10, findtime: 1.minute, bantime: 1.hour) do
    req.path == "/session" && req.post?
  end
end

# Custom response for throttled requests
Rack::Attack.throttled_responder = lambda do |_request|
  [429, { "Content-Type" => "text/plain" }, ["Rate limit exceeded. Retry later.\n"]]
end
```

## Redirect Security

```ruby
# BAD — open redirect
redirect_to params[:return_to]
redirect_to request.referer

# GOOD — validate against allowlist
ALLOWED_REDIRECTS = %w[/dashboard /profile /settings]
redirect_to params[:return_to] if ALLOWED_REDIRECTS.include?(params[:return_to])
```

## Missing Authentication

Every controller should require authentication unless explicitly public.

```ruby
# BAD — no authentication on destructive action
class AdminController < ApplicationController
  def destroy
    User.find(params[:id]).destroy
  end
end

# GOOD
class AdminController < ApplicationController
  before_action :authenticate_admin!

  def destroy
    User.find(params[:id]).destroy
  end
end
```
