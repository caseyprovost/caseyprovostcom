---
layout: post
title:  "Seven Days of Design Patterns | Part 3 - Form Objects"
date:   2019-05-11
categories: ruby,design-patterns
---

Welcome back to another episode of Design Patterns with yours truly. Today we are going to spend a little bit of time reviewing object-powered forms aka "Form Objects".

Let's set the stage by diving into a ficticious software team and observing how they work and react to requirements changing. AwesomePeople Inc is in the middle of launching their new social media platform. The last feature before they can ship is to create a user registration form. It is not quite that simple though, the business wants this site to be viral and has decided to only allow students with a **harvard.edu** email address to sign up. The team decides to add the validation to the model and ship it.

{% highlight ruby %}
class User < ApplicationRecord
  validates :name, :email, :username, :password
  validates :email, format: { with: /@harvard.edu/ }
end
{% endhighlight ruby %}

The site launches and is a smashing success on campus! Now the marketing team wants to expand to Yale, Stanford, and MIT. This is no problem for the dev team. They add a few conditionals to the `User` model and they ship it.

{% highlight ruby %}
class User < ApplicationRecord
  attr_accessor :school_suffix_requried

  SUPPORTED_EMAIL_DOMAINS = %w(
    harvard.edu
    standford.edu
    yale.edu
    mit.edu
  )

  validates :name, :email, :username, :password
  validates :email, format: { with: /@harvard.edu/ }, if: -> { school_suffix_requried == 'harvard.edu' }
  validates :email, format: { with: /@yale.edu/ }, if: -> { school_suffix_requried == 'yale.edu' }
  validates :email, format: { with: /@stanford.edu/ }, if: -> { school_suffix_requried == 'stanford.edu' }
  validates :email, format: { with: /@mit.edu/ }, if: -> { school_suffix_requried == 'mit.edu' }
  validates :email, inclusion: { in: SUPPORTED_EMAIL_DOMAINS }, if: -> { school_suffix_requried.blank? },
    message: 'domain is not currently supported'
end
{% endhighlight ruby %}

So far everyone is happy with the solutions the team has continued to ship. They are shipping fast, meeting requirements, and delighting users. This is when the marketing teams decides to make things interesting. Instead of expanding to new schools they want each school to be handled uniquely, as specified in the list below.

* For Harvard students, we need to send a "Harvard branded" email on signup
* For Stanford students, we need to send a "Stanford branded" email on signup and a text
* For Yale students, we need to send an email with no school branding
* For MIT students:
  - We need to hit an API they are providing to record the stats of who signed up
  - We need to send an "MIT branded" email on signup

Looking at the list of the requirements, Steve and the rest of the dev team begin to scratch their chins. They decide to add the functional code first and pair on improvements later.

{% highlight ruby %}
class User < ApplicationRecord
  attr_accessor :school_suffix_requried
  attr_accessor :mobile_number

  SUPPORTED_EMAIL_DOMAINS = %w(
    harvard.edu
    standford.edu
    yale.edu
    mit.edu
  )

  validates :name, :email, :username, :password
  validates :email, format: { with: /@harvard.edu/ }, if: -> { school_suffix_requried == 'harvard.edu' }
  validates :email, format: { with: /@yale.edu/ }, if: -> { school_suffix_requried == 'yale.edu' }
  validates :email, format: { with: /@stanford.edu/ }, if: -> { school_suffix_requried == 'stanford.edu' }
  validates :email, format: { with: /@mit.edu/ }, if: -> { school_suffix_requried == 'mit.edu' }
  validates :email, format: {
      with: /[@harvard.edu|@yale.edu|@stanford.edu|@mit.edu]/
    },
    if: -> { school_suffix_requried.blank? },
    message: 'domain is not currently supported'
  validates :mobile_phone_number, if: -> { school_suffix_requried == 'mit.edu' }
  validates :mobile_phone_number, if: -> { school_suffix_requried == 'mit.edu' }

  after_create :send_harvard_email, if: -> { school_suffix_requried == 'harvard.edu' }
  after_create :send_stanford_email, if: -> { school_suffix_requried == 'stanford.edu' }
  after_create :send_text, if: -> { school_suffix_requried == 'stanford.edu' }
  after_create :send_email, if: -> { school_suffix_requried == 'yale.edu' }
  after_create :send_mit_email, if: -> { school_suffix_requried == 'mit.edu' }
  after_create :send_stats_to_api, if: -> { school_suffix_requried == 'mit.edu' }

  private

  def send_text
    Twilio.sms_messages.create(number: mobile_phone_number, body: "Thank you for signing up with Amazing Inc!")
  end

  def send_harvard_email
    UserMailer.harvard_signup(self).deliver
  end

  def send_stanford_email
    UserMailer.standford_signup(self).deliver
  end

  def send_mit_email
    UserMailer.mit_signup(self).deliver
  end

  def send_email
    UserMailer.signup(self).deliver
  end

  def send_stats_to_api
    HTTParty.post('https://signup-stats.mit.edu/api', body: { email: user.email })
  end
end
{% endhighlight ruby %}

One of the newer devs on the team, Ben, reaches out to the team on Slack regarding the code. He says that this is starting to smell and will become really hairy as more and more schools are supported. He asks Steve to "pair up" to see what the two of them can clean up together. Before they get started, Ben suggests that they draft some general requirements and go from there. Here is what they came up with:

* Users need to be able to sign up based on school/domain-name
* Users sometimes need tailored/branded emails upon signup success
* Users sometimes need unbranded/standard emails upon signup success
* Users sometimes need to receive text messages upon signup success
* For certain users they need to send API calls after a successful signup

As they list this all out it starts to become clear that they have two general things to account for. First, they need to make sure they have valid data in each case...meaning the standard fields and whatever nuanced ones the customers require (like cell number). The second general requirement is support for custom "success handling" for each school.

Steve suggests extracting out the school concept into an object and proposes it like so:

{% highlight ruby %}
class School < ApplicationRecord
  has_many :users

  # The fields:
  # name: string | General identifier like "Yale" or "Stadford"
  # email_domain: string | email suffix such as "mit.edu"
  # slug: friendly identifer such as "brown-university"
  # branded_email: boolean
  # signup_email: boolean
  # signup_text: boolean
end
{% endhighlight ruby %}

{% highlight ruby %}
class User < ApplicationRecord
  attr_accessor :mobile_number

  belongs_to :school

  validates :name, :email, :username, :password
  validates :mobile_phone_number, if: -> { school&.signup_text? }
  validates :validate_email_domain_for_school, if: -> { email.present? && school.present? }
  validates :validate_email_for_supported_schools, if: -> { email.present? && school.nil? }

  after_create :send_branded_email, if: -> { school&.signup_email? && school&.branded_email? }
  after_create :send_standard_email, if: -> { school&.signup_email? && !school&.branded_email? }
  after_create :send_text, if: -> { school&.signup_text? }
  after_create :send_stats_to_api, if: -> { school_suffix_requried == 'mit.edu' }

  private

  def validate_email_domain_for_school
    pattern = "/@#{school.email_domain}/"
    return if email.match?(pattern)
    errors.add(:email, "is not currently supported")
  end

  def validate_email_for_supported_schools
    valid_domains = School.pluck(:email_domain).map{ |domain| "@#{domain}" }
    pattern = "/[#{valid_domains.join('|')}]/"
    return if email.match?(pattern)
    errors.add(:email, "is not currently supported")
  end

  def send_text
    Twilio.sms_messages.create(number: mobile_phone_number, body: "Thank you for signing up with Amazing Inc!")
  end

  def send_branded_email
    UserMailer.branded_signup(self, school).deliver
  end

  def send_standard_email
    UserMailer.signup(self, school).deliver
  end

  def send_stats_to_api
    HTTParty.post('https://signup-stats.mit.edu/api', body: { email: user.email })
  end
end
{% endhighlight ruby %}

Ben is getting stoked. The signup emails and texts definitely seem to be handled in a more straight forward and standard way. Not only that, but adding support for additional schools could now be as simple as adding a new row in the database. That is pretty valuable and really enables the marketing team to move quicker. Not only that, but this move could also allow the DRYing up of the controllers and view code!

A few things are still bothering him though. That pesky `send_stats_to_api` callback on user is making testing more complicated than it should be. Not only that, but it is a special case that doesn't seem to apply to other schools. It seems that sometimes schools want to introduce requirements that other schools may never use. Last of all..testing the user model is still complicated and involves stubbing out API calls, mailers, etc.

As Ben and Steve are pairing, Ben shares that he heard of a pattern called "Form Objects" which help with issues like this. Mainly, separating UI/UX behaviors and needs from data models. Steve wasn't aware of this pattern and decides to let Ben drive the session while he observes and learns. Here is what Ben produces:

{% highlight ruby %}
class UserSignupForm
  include ActiveModel::Model
  include ActiveModel::Attributes

  attribute :school, School
  attribute :email, String
  attribute :mobile_phone_number, String
  attribute :name, String
  attribute :username, String
  attribute :password, String

  validates :name, :email, :username, :password
  validates :mobile_phone_number, if: -> { school&.signup_text? }
  validates :validate_email_domain_for_school, if: -> { email.present? && school.present? }
  validates :validate_email_for_supported_schools, if: -> { email.present? && school.nil? }

  def save
    if valid?
      persist!
    else
      false
    end
  end

  private

  def validate_email_domain_for_school
    pattern = "/@#{school.email_domain}/"
    return if email.match?(pattern)
    errors.add(:email, "is not currently supported")
  end

  def validate_email_for_supported_schools
    valid_domains = School.pluck(:email_domain).map{ |domain| "@#{domain}" }
    pattern = "/[#{valid_domains.join('|')}]/"
    return if email.match?(pattern)
    errors.add(:email, "is not currently supported")
  end

  def user_attributes
    attributes.slice(:school, :email, :name, :username, :password, :mobile_phone_number)
  end

  def persist!
    user = User.new(user_attributes)

    ApplicationRecord.transaction do
      if user.save
        SignupUser.new(user).call
      else
        flush_errors_to_form(user)
      end
    end
  end

  def flush_errors_to_form(user)
     user.errors.each_pair do |field, messages|
      error_field = respond_to?(field) ? field : :base
      messages.each { |message| errors.add(error_field, message) }
    end
  end
end
{% endhighlight ruby %}

And he creates the complimentary "Service" for the user signup process:

{% highlight ruby %}
class SignupUser
  def initialize(user, school)
    @user = user
    @school = school
  end

  def call
    send_branded_email
    send_standard_email
    send_text
    send_stats_to_api
    true
  end

  private

  def send_text
    if @school&.signup_text?
      Twilio.sms_messages.create(number: mobile_phone_number, body: "Thank you for signing up with Amazing Inc!")
    end
  end

  def send_branded_email
    if @school&.signup_email? && @school&.branded_email?
      UserMailer.branded_signup(@user, @school).deliver
    end
  end

  def send_standard_email
    if @school&.signup_email? && !@school&.branded_email?
      UserMailer.signup(@user, @school).deliver
    end
  end

  def send_stats_to_api
    if @school&.slug == 'mit'
      HTTParty.post('https://signup-stats.mit.edu/api', body: { email: @user.email })
    end
  end
end
{% endhighlight ruby %}

Finally, Ben cleans up the User model.

{% highlight ruby %}
class User < ApplicationRecord
  belongs_to :school, optional: true

  validates :name, :email, :username, :password
end
{% endhighlight ruby %}

Steve has taken all this change in and has been mulling it over. A few seconds after Ben finishes his refactor he asks about the complexity and overhead of additional objects. Ben is more than thrilled to be able to teach Steve the benefits of this pattern and so he lists out what this refactor achieved:

* Creating a user model in tests or elsewhere in the codebase no longer triggers emails, texts, or API calls
* Changing form error messages or fields in the UI simply translate to alterations in the one form model processing that concern
* Handling sign up success functionality in one spot that is easily changed allows the team to handle that concern with surgical precision. There is only one place that logic lives.

After a few moments Steve's eyes brighten up and he starts to get excited. In the "Admin" section of the site there is an ability to create users for AwesomePeople Inc. When these users are added, no side-effects should happen. Meaning: no emails, texts, API calls, or otherwise should be triggered..regardless of if they are assigned a school or not.

The scene fades with a close up of Steve and Ben's monitor. They are going through all of their tests involving users and deleting the stubbed out email, text, and API logic from their previous implementation.

Isn't it a wonderful feeling to delete code? Or how about removing side-effects from your tests, making them simpler and easier to reason about and change? That is really what the form object pattern is about. It is about capturing the UI needs of a form. Typically, one that needs to save more than one object at a time or when there are special processing needs; like we saw in the story above.

I hope you enjoyed the story and maybe even found a solution to a personal pain in a project you are currently on. Til next week's design pattern session, see you soon!
