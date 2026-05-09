# Refactoring: Replace Callback with Method

## Context
An ActiveRecord callback (`after_create`, `after_save`, etc.) that performs
work unrelated to the record's own persistence — sending email, creating records
in other models, calling external services. The side effect is invisible to the
caller and fires even when triggered by unrelated saves.

See also: `coding-classic.md § Callbacks` for when callbacks are appropriate
(own-state changes, async dispatch via jobs), and `shared/callbacks.md` for the
scoring system and extraction patterns.

## Before
```ruby
class Invitation < ApplicationRecord
  after_create :deliver

  private

  def deliver
    InvitationMailer.invite(self).deliver_later
  end
end

# Caller has no visibility into delivery:
Invitation.create!(email: email, survey: survey)
```

## After
```ruby
class Invitation < ApplicationRecord
  # No callback — deliver is a public method, called explicitly
  def deliver
    InvitationMailer.invite(self).deliver_later
  end
end

class SurveyInviter
  def create_and_deliver_invitations
    invitations = parsed_emails.map do |email|
      Invitation.create!(email: email, survey: @survey)
    end
    invitations.each(&:deliver)
  end
end
```

## Why
Delivery is visible at the call site. If a `create!` fails mid-loop, no emails
are sent for records that don't exist yet. The callback can't be accidentally
triggered by an unrelated `save` elsewhere. Delivery is testable without
persistence side effects.

## Alternative: Event Bus

When multiple subscribers need to react to the same lifecycle event, use
the event bus instead of extracting individual methods:

```ruby
# Rails 8.1+ — Rails.event (in-process, no external backend)
class Invitation < ApplicationRecord
  after_commit on: :create do
    Rails.event.notify("invitation.created", { id: id, survey_id: survey_id })
  end
end

# Rails ≤ 8.0 — ActiveSupport::Notifications
ActiveSupport::Notifications.instrument("invitation.created", invitation: self)
```

See `refactorings/extraction-signals.md § Event-driven extraction` for
subscriber setup and ops notes, and `shared/instrumentation.md` for the full
event pipeline reference.

## When NOT to apply
- The callback updates derived state on the same record (`before_save :normalize_email`,
  `before_save :update_search_index, if: :title_changed?`) — own-state
  management is the right use of callbacks.
- The callback dispatches to a background job (`after_create_commit :notify_later`)
  — async dispatch is an acceptable callback use; the job itself should be thin.

For multi-channel notifications extracted from callbacks, see `shared/notifications.md`.

_[Ruby Science →](https://thoughtbot.com/ruby-science/replace-callback-with-method.html)_
