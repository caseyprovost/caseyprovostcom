---
layout: post
title:  "7 Days of Design Patterns | Part 1"
date:   2015-02-12 13:46:40
categories: wanderings
---
Welcome! I am so glad you came here to read about design patterns. With the Ruby on Rails community having such
an abundance of newer developers and newer self-taught developers I thought it was excellent time to share some
real world usage of design patterns that have helped save dozens of applications I have worked on and simplified
many bugs and issues.

Today day 1 is going to be all about service objects, and we will talk about them again as the series continues. You
can expect to read about organizers, form objects, query objects, presenters, the adapter pattern, and factories. There
are definitely more patterns out there to read about and people like Martin Fowler or Robert C. Martin, but these are a
nice start and can be used right away on most RoR codebases.

What I help you to accomplish is to gain a rudimentary understanding of these patterns and some ways to apply them in
good ways that bring up the health of your code instead of adding complexity. I suppose one of the biggest caveats to
design patterns is that they are applied improperly every day and they are just tools that solve known problems in programming.
If you think about them critically and use them judiciously however I am sure you will be just fine.

Alright, let's dive in.

### These are the droids you are looking for

Most education starts with a good definition, so let's start there. As I understand it *a Service Object implements the
systems interactions and usually involves more than one model*. What does this mean? Well in the simplest forms it means
that service objects are classes that achieve a discrete business or domain objective. Things like registering a user,
syncing an object with an API, or maybe applying tags to an article based on some machine learning analysis. Lets take a
piece at the first usage; implementing a user's actions.

{% highlight ruby %}
class RegisterUser
  def initialize(params)
    @params = params
  end

  def call(&block)
    @user = User.create!(@params)
    send_user_to_firebase(@user)
    send_user_intercom(@user)

    if block_given?
      block.call(@user)
    else
      true
    end
  end

  private

  def send_user_to_fire_base(user)
    # Sends the user to the Firebase service
  end

  def send_user_intercom(user)
    # Sends the user to the Intercom service
  end
end
{% endhighlight %}

As you can see, registering a user can get pretty complicated. If you have tried putting all of this in a controller like I used
to then you know how hard it is to test some of this stuff. You also may have found out that it is simply not very usable in things
like an API user registration controller or when setting up an admin. This allows us to encapsulate what it means to register a user
in a clean and reusable way. For fun, you may have noticed the block being passed to the #call method. This is a little trick I user
all the time to execute custom code after a service succeeds because it is NOT reusable. Let's take a quick look at a controller's use.

{% highlight ruby %}
class UserController < ApplicationController
  def create
    service = RegisterUser.new(params[:user])
    @user = user

    service.call do |user|
      @user = user
      UserMailer.welcome_email(user).deliver_now!
    end

    if @user
      flash[:success] = "Thank you for signing up!"
      redirect_to user_email_confirmation_path(@user)
    else
      render :new
    end
  end
end
{% endhighlight %}

As you can see from this example the block to send an email was utilized so we could set an instance variable for a particular use
case and more specifically to send an email to the user. There are so many times that sending email based on one action is not
reusable, so this is a spiffy little trick to help fix particular issue.

The next and finally application is using a service object to perform a system job or operation. Let's look at a class as an
example that is responsible for syncing a user with Quickbooks.

{% highlight ruby %}
class SyncUserWithQuickbooks
  def initialize(user)
    @user = user
  end

  def call
    return if @user.last_updated_with_qb_at < quickbooks_user.last_change_at

    success = Quickbooks::Customers.update(quickbooks_user, @user.quickbooks_attributes)

    if success
      @user.last_updated_with_qb_at = Time.zone.now
      @user.save!
      true
    else
      false # let the client know the sync failed.
    end
  end

  private

  def quickbooks_user
    @quickbooks_user ||= Quickbooks::Customers.find(@user.quickbooks_id)
  end
end
{% endhighlight %}

While both of these classes are fairly similar (you will note the same public interface of #initialize and #call), this one performs
a job of the system while the former performs a user's action. In my experience both of these are fairly reusable, however reuse is not necessarily the only reason to create service classes. They come in handy to encapsulate a business or piece of domain logic that is in
one isolated place that you can iterate on as time flies by. And you can best believe you will either be removing these classes,
drastically modifying them, or even making them part of an Organizer (for discussion next time) in order to accomplish the same basic
concept.

### The dark side of the force is so seductive
And you thought you were only going to read one Star Wars related subtitle...Anyway, it is important to point out that these patterns
all come with disclaimers and what I find to be some best practices as well.

#### Manage Those Transactions
If you service objects persists more than 1 model, then please for the love of all that is holy wrap that stuff in a transaction. You
may or may not care now, but I promise you will later. It will be on one of those dark nights where you are looking at the DB and notice
lots of rogue records or maybe even when a user can't register because their user was saved but the complimentary sync to Firebase failed.
Transactions will go a long way to save you from producing bad data you have to cleanup later.

#### One Size Does NOT Fit all
If you find that your service object does a few things or even many or perhaps just calls other service objects that perform business
logic, then you probably don't need a service object. You are probably looking for an organizer, but we will talk about that next time. For
now just hold your breath and try to wait patiently.

#### We Need An Injection... Stat
For your sanity and easy of testing, pass objects into your service classes instead of doing lookups. This way you are not prohibited from
using duck-typed classes and your in tests you can easily mock those arguments. This will save you a load of trouble and boilerplate stubbing of ActiveRecord lookups.

#### You Must Unlearn What You Have Learned
When you jumped into Rails you learned all about MVC and everything fit in one of those layers. It was just too easy to throw domain
logic on models via quick little methods, and it was so slick to right that 60 line search method in your controller. Some of these patterns
will make sense, and others will challenge your idea of what it means to develop a clean Rails app. You may find that you have a services
directly in your application that becomes just as big as your models or controllers directory. This is ok. It means that you are trimming
the fat from those controllers and models and instead are creating isolated, reusable, pieces of domain logic that are far easier to test
and maintain.

### That's a Wrap
Thanks so much for hanging in there with me through my pandering and meandering. In my next article we will bite from the coding tree of
knowledge and pluck the ever-so-delicious organizer fruit from a low hanging branch. I won't tease you now, but seriously they are awesome. See you soon!

PS: If you want to troll me or send me your thoughts on this article feel free to [email me](mailto:caseyr.provost@gmail.com)
