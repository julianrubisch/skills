# Security Review Reference

Diagnostic layer — what to flag, how severe, and why. Prescriptive patterns
(how to write secure code) live in shared/security.md.

## Anti-patterns

### SQL Injection via String Interpolation

**Severity: Critical**

```ruby
# BAD
User.where("name = '#{params[:name]}'")

# GOOD
User.where(name: params[:name])
```

**Signal:** String interpolation inside `.where(...)`, `.order(...)`,
`.select(...)`, or any raw SQL call.

### Mass Assignment Without Strong Parameters

**Severity: High**

```ruby
# BAD
User.update(params[:user])

# GOOD
User.update(user_params)  # via permit(...)
```

**Signal:** `params[:model]` passed directly to AR methods without `.permit`.

### Missing Authorization Scope

**Severity: High**

**Problem:** Records looked up by ID alone without verifying ownership.
Any authenticated user can access any record.

```ruby
# BAD
@card = Card.find(params[:id])

# GOOD
@card = Current.account.cards.find(params[:id])
```

**Signal:** Bare `Model.find(params[:id])` in controllers without a
preceding scope or Pundit `authorize` call.

### Calling `.html_safe` on User Input

**Severity: High**

Any string from user input, the database, or an external API must be
escaped before `.html_safe`. Grep for it in every PR.

**Signal:** `.html_safe` on a string that wasn't produced by a Rails helper.

### Missing Content Security Policy

**Severity: Medium**

**Signal:** No CSP initializer, or a permissive one with `unsafe-eval`
or wildcard `*` sources.

### Unprotected External URL Fetching

**Severity: High**

**Signal:** `Net::HTTP`, `HTTParty`, `Faraday`, or `open-uri` called with
a user-supplied URL without DNS pinning or IP range checks.

**Fix:** See SSRF pattern in shared/security.md.

## Heuristics

- Every `find(params[:id])` without a scope is a potential IDOR — flag it
- `.html_safe` is a code smell in PRs — verify each one
- CSP violations in logs are worth investigating, not dismissing
- Check ActionText allowed_tags in any app using rich text

<!-- Add your own security review heuristics below -->
