---
layout: post
title:  "Seven Days of Design Patterns | Part 4 - Adapter Pattern"
date:   2019-05-30
categories: ruby,design-patterns
---

Recently I ran into a rather interesting problem. The app I was working on needed to add support for multiple labs. The application is one that helps folks order medical tests and get them to a nearby lab without all the hoops involved of going through a doctor or doctors. For many years there was just one lab provider, let's call them Legacy Labs. However, they have decided to shut down all of their integrations so now our application needs to support at least one new lab provider. Let's call this lab, Future labs.

Of course we can't just perform hard cut over. We need a way for both lab providers to co-exist for at least some span of time. This enables customers to continue to receive lab results from the primary/old lab provider. Now the question is: "how on earth do that". Let's tackle this one problem at a time. The easiest way to do that is to list out the pieces.

* Allow users to find a lab location belonging to either provider
* Allow users to receive lab tests results from either provider

For the sake of aruments let's say that all the types of labs and details are the same. That way we can get right into the nitty gritty.

Alright so let's tackle the first problem of locating a lab provider. Currently the class looks something like this.

{% highlight ruby %}
class LegacyLabLocations
  def initialize(zip_code)
    @zip_code = zip_code
  end

  def fetch
    @results = []

    begin
      @results = HttParty.get('https://legacy-labs.com/locations', query: { zip_code: @zip_code })
    rescue => e
      Rails.logger.error(e)
    end

    @results
  end
end
{% endhighlight %}
