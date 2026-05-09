# Refactoring: Introduce Form Object

## Context
A controller action that validates input, orchestrates multiple models, or
contains conditional logic beyond basic CRUD. Also appropriate when validation
rules belong to one UI flow and would pollute the domain model.

See also: `patterns.md § Form Objects` for the preferred structure.

## Before
```ruby
class InvitationsController < ApplicationController
  def create
    @emails = params[:emails].split(",").map(&:strip)
    if @emails.empty?
      flash[:error] = "Provide at least one email"
      redirect_to new_invitation_path and return
    end
    @emails.each do |email|
      Invitation.create!(email: email, survey: @survey)
      InvitationMailer.invite(email, @survey).deliver_later
    end
    redirect_to survey_path(@survey)
  end
end
```

## After
```ruby
# app/forms/survey_inviter.rb
class SurveyInviter
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :emails_raw, :string
  attribute :survey

  validates :emails_raw, presence: true
  validate :emails_are_valid

  def save
    return false unless valid?
    parsed_emails.each do |email|
      Invitation.create!(email: email, survey: survey)
      InvitationMailer.invite(email, survey).deliver_later
    end
    true
  end

  private

  def parsed_emails = emails_raw.to_s.split(",").map(&:strip)

  def emails_are_valid
    parsed_emails.each do |e|
      errors.add(:emails_raw, "#{e} is not valid") unless e.include?("@")
    end
  end
end

class InvitationsController < ApplicationController
  def create
    @inviter = SurveyInviter.new(inviter_params.merge(survey: @survey))
    if @inviter.save
      redirect_to survey_path(@survey)
    else
      render :new, status: :unprocessable_entity
    end
  end
end
```

## Why
The controller handles only request/response. The form object is independently
testable and reusable, and supports custom validations without coupling them to
the domain model.

## When NOT to apply
- Single-model form with standard validations — use the model directly.
- The controller action is thin and the only caller — the abstraction may not pay off.

_[Ruby Science →](https://thoughtbot.com/ruby-science/introduce-form-object.html)_
