---
layout: post
title:  "Seven Days of Design Patterns | Part 5 - Query Objects"
date:   2019-06-30
categories: ruby,design-patterns
---

Welcome back folks! I hope you have been enjoying the series on design patterns so far. Today I wanted to carve our some time to talk about **query objects**. Now you might say something like, "but ActiveRecord already has scopes!"..and you would be 100% correct. The problem is in the ease of use of scopes and their ability to chain. Take the following example:

{% highlight ruby %}
class IncomeReportController < ApplicationController
  def show
    @transactions = Transaction
                      .completed
                      .past_90_days
                      .not_refunded
                      .joins(:user)
                      .where(user: { status: 'active' })
  end
end
{% endhighlight %}

Do you see? It's almost too easy to take all of these utility scopes and chain them to create some powerful query. While there is nothing inheritantly wrong in chaining scopes, there is a price paid. In cases like this the price is lack of meaning and difficulty in tests. By "lack of meaning" what I'm really trying to say is that it's unclear why this is the correct query. Let's explore this a little by pulling out a query object.

{% highlight ruby %}
module Report
  class IncomeTransactionsQuery
    attr_reader :relation

    def initialize(relation)
      @relation = relation
    end

    def resolve
      relation.completed.not_refunded.joins(:user).where(user: { status: 'active' })
    end
  end
end
{% endhighlight %}

I know all we did was extract the query logic, but let's take a look at the controller now.

{% highlight ruby %}
class IncomeReportController < ApplicationController
  def show
    all_transactions = query_klass.new(Transaction.past_90_days).resolve
    @transactions = all_transactions.page(params[:page]).per_page(params[:per_page])
  end

  private

  def query_klass
    Report::IncomeTransactionsQuery
  end
end
{% endhighlight %}

We can clearly see that a collection is being fetched specifically for our use case of reporting income. That is pretty cool. This also gives us only one place to change if the requirements for this report change. Not sold yet? Let's add some requirements! After all, we are building real world applications and concepts like reports change often. Here are the new requirements:

* All existing conditions, except the date range, are solid
* The report needs to allow filtering by a start and end date
* The report needs to allow filtering by the status of a customer
* The report needs to allow filtering by a minimum amount of the transaction
* The report needs to allow filtering by user_id

Wowza! Leave it up to the finance team to add complexity to our simple little report. But do not fear, our query object is here :)

{% highlight ruby %}
module Report
  class IncomeTransactionsQuery
    attr_reader :relation
    attr_reader :filters

    def initialize(relation:, filters: {})
      @relation = relation
      @filters = filters
    end

    def resolve
      scope = relation.completed.not_refunded
      return scope.distinct if filters.empty?

      scope = apply_filter(scope, :start_date)
        .yield_self { |scope| apply_filter(scope, :end_date) }
        .yield_self { |scope| apply_filter(scope, :user_status) }
        .yield_self { |scope| apply_filter(scope, :user_id) }
        .yield_self { |scope| apply_filter(scope, :amount) }
        .yield_self { |scope| scope.distinct }
    end

    private

    def apply_filter(scope, filter)
      return scope if filters[filter].nil?
      send("filter_by_#{filter}", scope, filters[filter])
    end

    def filter_by_start_date(scope, value)
      scope.where(created_at: value.to_datetime.beginning_of_day)
    end

    def filter_by_end_date(scope, value)
      scope.where(created_at: value.to_datetime.end_of_day)
    end

    def filter_by_user_status(scope, value)
      scope.joins(:user).where(user: { status: value })
    end

    def filter_by_amount(scope, value)
      scope.where('amount >= ?', value)
    end

    def filter_by_user_id(scope, value)
      scope.joins(:user).where(user: { id: value })
    end
  end
end
{% endhighlight %}

Our little query object sure grew up fast! It's now responsible for more than the original static concept of a query for the income report. It also knows how to filter that query. Even though that's true, I'd argue it doesn't violate the [Single responsibility principle](https://en.wikipedia.org/wiki/Single_responsibility_principle). The only responsibility our query object has is to return the data needed for an income report. Let's move onto the controller and see how these changing requirements impacted that code.

{% highlight ruby %}
class IncomeReportController < ApplicationController
  def show
    all_transactions = query_klass.new(
      relation: Transaction.all,
      filters: filters
    ).resolve

    @transactions = all_transactions.page(params[:page]).per_page(params[:per_page])
  end

  private

  def filters
    params[:filter] ||= {}
    params.require(:filter).permit(
      :start_date,
      :end_date,
      :user_id,
      :user_status,
      :amount
    )
  end

  def query_klass
    Report::IncomeTransactionsQuery
  end
end
{% endhighlight %}

Well look at that. Our controller doesn't change that much after all. The only thing we need to do is accept filter parameters and pass them to the query object. Not to shabby. Before I leave you to try this pattern on your own, I'll enumerate a few good use-cases I have found for query objects:

* When there are complex queries or scope chaining
* When the query needs filtering that is not re-used
* When the scope/query interacts with more models then itself

And here are some rules I apply when creating this query objects:

* Accept a scope as the first argument (scope, relation, etc) are all good names
* Stick to a naming convention (DoSomethingQuery, OtherThingQuery, etc)
* Use a Base query object to extract away the common interface (attribute readers, initialize, etc)
* Always return an ActiveRecord scope
* Define *.resolve* as a class method that accepts the same arguments as `initialize` (helps with testing)

There are a lot if little details we kind of glossed over here. Details such as the littly bit of meta-programming in the query object for applying filters, the usage of `yield_self`, and the lack of data santization on the filters themselves. These are the things I leave to you, the readers, to explore yourselves. What you see here is how I approach these problems in real world RoR applications servicing hundreds to thousands of users on a daily basis. This pattern has been relatively easy to maintain and has stood the test of time on numerous RoR applications as old as eleven years. Don't just take my word for it though. Try it out in your application(s) and see how it goes.

Next week, instead of moving onto the next design pattern (Presenters) we will be covering how to test the code you saw above. So buckle up and get ready for some test slinging!

As always, thank you for taking the time to read my ramblings. You truly are the best!
