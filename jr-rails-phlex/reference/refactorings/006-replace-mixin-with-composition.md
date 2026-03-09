# Refactoring: Replace Mixin with Composition

## Context
A mixin (concern) that contains business logic that's hard to test in isolation,
causes name clashes when included in multiple classes, or whose methods show
feature envy toward an external collaborator.

Note: purely organizational concerns (associations, delegates, scopes) are fine
as mixins — this refactoring targets logic-heavy ones.

## Before
```ruby
module Inviter
  def render_message(template)
    ApplicationController.render(template, assigns: { invitation: self })
  end
end

class EmailInviter
  include Inviter
  def invite(email) = deliver(email, render_message(:email_invitation))
end

class MessageInviter
  include Inviter
  def invite(phone) = send_sms(phone, render_message(:sms_invitation))
end
```

## After
```ruby
class InvitationMessage
  def initialize(invitation, template)
    @invitation = invitation
    @template   = template
  end

  def render
    ApplicationController.render(@template, assigns: { invitation: @invitation })
  end
end

class EmailInviter
  def invite(email)
    message = InvitationMessage.new(self, :email_invitation).render
    deliver(email, message)
  end
end

class MessageInviter
  def invite(phone)
    message = InvitationMessage.new(self, :sms_invitation).render
    send_sms(phone, message)
  end
end
```

## Why
`InvitationMessage` is testable without including a module into a host class.
Dependencies are explicit. No name clash risk. Move one method at a time for a
safe migration path.

## When NOT to apply
- The concern is purely declarative (`has_one`, `delegate`, `scope`) — mixins
  are fine for horizontal declarations with no logic.
- The composed class would immediately show feature envy back at the including
  class — the boundary is in the wrong place.

_[Ruby Science →](https://thoughtbot.com/ruby-science/replace-mixin-with-composition.html)_
