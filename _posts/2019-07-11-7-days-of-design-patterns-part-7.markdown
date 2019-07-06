---
layout: post
title:  "Seven Days of Design Patterns | Part 7 - Proxy Objects"
date:   2019-07-11
categories: ruby,design-patterns
---

Welcome and welcome back! Today I have delicious little code morsel for you. Proxy objects are a simple pattern you can apply to make objects more secure. At least, that is where I find them to be the most valuable. Now let's begin our adventure together.

Say you are working in a bank. But not just any bank... a bank on Rails! The first task you are given is to secure the existing bank account object that is floating around the system. The requirements are pretty simple..any class that works with a bank account object needs to lose access to the account number, routing number, and customer mailing address information.

While this new requirement is a little annoying and probably means swapping objects and around as well as tests...let's zoom in on just one example of how we might solve this with a proxy. Let's start by defining the current interface of a bank account.

{% highlight ruby %}
# our existing bank account class
class BankAccount < ApplicationRecord
  # has account_number
  # has routing_number

  def customer_address
    Address.from_customer(customer)
  end
end
{% endhighlight %}

This looks pretty innocent, but we know it has access to all the columns on the bank_accounts table in database. Let's jump over to a controller where BankAccounts are used.

{% highlight ruby %}
class BankAccountsController
  def show
    @bank_account = current_customer.bank_accounts.find(params[:id])
  end
end
{% endhighlight %}

This method on the controller should be simple enough to swap out. Let's create our proxy!

{% highlight ruby %}
class BankAccountProxy
  def initialize
    @bank_account = bank_account
  end

  def name
    @bank_account.name
  end

  def customer_name
    @bank_account.customer.name
  end
end
{% endhighlight %}

And there it is. A little btye sized object that is easy to test and use to replace our usages of BankAccount throughout the system. Of course you may have to pull more methods into the proxy and create proxy collections in areas where bank_accounts are listed...but I'll leave those circumstances to your imagination. Time to perform Operating Swap!

{% highlight ruby %}
class BankAccountsController
  def show
    raw_bank_account = current_customer.bank_accounts.find(params[:id])
    @bank_account = BankAccountProxy.new(raw_bank_account)
  end
end
{% endhighlight %}

Boom, that's it. Now the bank accounts view can be trimmed back to not expose that pesky sensitive data. Even better, in our tests for this view, we will see exceptions where the insecure attributes are accessed. This makes fixing the view and meeting our requirements of removing sensitive data as easy as pie.

Whew! That was a longer seven weeks than I thought it was going to be. There is so much more we could talk about. However, I'll stop here and next week we will go back to random useful postings. Just to tease, I am working on a rather large project that I hope to start sharing soon. The project is a bookstore built on Rails with [Graphiti](https://www.graphiti.dev/guides/), microservices, React/Apollo, all tied together with [GraphQL](https://graphql.org/) proxy.

If this project or any moving piece would interest you to read about, drop a comment below. I'll also be polling on [Twitter](https://twitter.com/caseyprovost/status/1147566716031029248). Whatever is upvoted the most will be the subject of the next blog post!
