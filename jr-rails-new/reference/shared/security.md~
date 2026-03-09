# Security: Shared Reference

Prescriptive guide — how to write secure Rails code. Loaded by both coding
skills and review agents. Review-specific concerns (what to flag, severity)
live in review-security.md.

## XSS Prevention

Escape user content before rendering. Never call `.html_safe` on user-supplied
strings directly.

```ruby
# GOOD — escape first, then mark safe
def formatted_content(text)
  simple_format(h(text)).html_safe
end
```

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

## ActionText Sanitization

Restrict allowed tags explicitly — don't rely on ActionText defaults.

```ruby
# config/initializers/action_text.rb
Rails.application.config.after_initialize do
  ActionText::ContentHelper.allowed_tags = %w[
    strong em a ul ol li p br h1 h2 h3 h4 blockquote
  ]
end
```

## Strong Parameters

Always whitelist at the controller boundary. Never pass `params` directly
to model methods.

```ruby
def user_params
  params.require(:user).permit(:name, :email)
end
```

## Authorization Scoping

Scope all record lookups to the current user or account. Never look up
a record by ID alone without verifying ownership.

```ruby
# BAD — any authenticated user can access any record
@card = Card.find(params[:id])

# GOOD — scoped to current account
@card = Current.account.cards.find(params[:id])
```

Use Pundit policies for non-trivial authorization logic (see patterns.md).

## SQL Injection

Use parameterized queries. Never interpolate user input into SQL strings.

```ruby
# BAD
User.where("name = '#{params[:name]}'")

# GOOD
User.where(name: params[:name])
User.where("name = ?", params[:name])
```
