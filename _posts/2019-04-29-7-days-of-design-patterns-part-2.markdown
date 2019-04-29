---
layout: post
title:  "Seven Days of Design Patterns | Part 2 - Organizers"
date:   2019-04-29
categories: ruby,design-patterns
---

I recently rolled off of a large monolithic application, written in Rails, that has been running for twelve years! As I sat down to write about our next pattern, Organizers, I realized that most often I have used them to solve problems in these large applications where there is often more code re-use as well as complexity. Let's take a look at a real world use case. For this exercise we will be using the [Interactor gem](https://github.com/collectiveidea/interactor).

Let's get started.

{% highlight ruby %}
class RegisterUser
  include Interactor::Organizer

  organize CreateUser, RecordUserData, WelcomeNewUser
end
{% endhighlight %}

Can you believe this class used to be 200+ lines of code with nested conditionals, explicit rollbacks, and even API calls to third party services? I bet you can, but that's not the point. Just look at that clean and minimal code! Using and testing this class is exceptionally easy.

{% highlight ruby %}
class RegistrationsController < ApplicationController
  def create
    result = RegisterUser.call(user_params: user_params)

    if result.success?
      redirect_to dashboard_path
    else
      @user = result.user
      render :new
    end
  end

  private

  def user_params
    params.require(:user).permit!
  end
end
{% endhighlight %}

Here is a link to the Github code where you can see how these interactors and organizers are built and tested: [github](https://github.com/caseyprovost/interactors-exercise).

Ok ok, so you you recognized that perhaps this code is a little... over-engineered. If this was the only time in your codebase these objects were used, I'd likely agree. However, let's imagine the business introduced two more requirements. First, they want to introduce a marketing effort where affiliates can put up a registration form for our application. This form needs to register new users and send a welcome email from the affiliate.

I wonder if we can reuse our interactors here...Let's start with the organizer.

{% highlight ruby %}
class RegisterAffiliateUser
  include Interactor::Organizer

  organize CreateUser, RecordUserData, WelcomeAffiliateUser
end
{% endhighlight %}

That wasn't too hard right? Now let's modify CreateUser to meet the requirement.

{% highlight ruby %}
class CreateUser
  include Interactor

  def call
    user = User.new(user_params)

    if user.save
      context.user = user
    else
      context.fail!(message: "Could not save user")
    end
  end

  private

  def user_params
    context.slice(:email, :password, :name, :affiliate_id)
  end
end
{% endhighlight %}

The marketing team isn't quite done though. If a user registers for our application in this way, then the affiliate needs to receive $5.00 from us as a referral reward.

{% highlight ruby %}
class RegisterAffiliateUser
  include Interactor::Organizer

  organize CreateUser, RewardAffiliate, RecordUserData, WelcomeAffiliateUser
end
{% endhighlight %}

Do you see now? We have created small interactors that execute specific tasks. These are small enough that we can compose them in Organizers that encapsulate complex processes in our domain. This is the true power of the Organizer pattern when used in conjunction with the Interactor/Service pattern.
