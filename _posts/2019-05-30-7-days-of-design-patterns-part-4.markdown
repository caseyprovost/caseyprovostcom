---
layout: post
title:  "Seven Days of Design Patterns | Part 4 - Adapter Pattern"
date:   2019-05-29
categories: ruby,design-patterns
---

Recently I ran into a rather interesting problem. The app I was working on needed to add support for multiple labs. The application is one that helps folks order medical tests and get them to a nearby lab without all the hoops involved of going through a doctor or doctors. For many years there was just one lab provider, let's call them Legacy Labs. However, they have decided to shut down all of their integrations so now our application needs to support at least one new lab provider. Let's call this lab, Future labs.

Of course we can't just perform hard cut over. We need a way for both lab providers to co-exist for at least some span of time. This enables customers to continue to receive lab results from the primary/old lab provider. Now the question is: "how on earth do that". Let's tackle this one problem at a time. The easiest way to do that is to list out the pieces.

* Allow users to find a lab location belonging to either provider
* Allow users to receive lab tests results from either provider

For the sake of aruments let's say that all the types of labs and details are the same. That way we can get right into the nitty gritty.

Alright so let's tackle the first problem of locating a lab provider. Currently the class looks something like this.

{% highlight ruby %}
class LabLocations
  def initialize(zip_code)
    @zip_code = zip_code
  end

  def fetch
    @results = []

    begin
      @results = HTTParty.get('https://legacy-labs.com/locations', query: { zip_code: @zip_code })
    rescue => e
      Rails.logger.error(e)
    end

    @results
  end
end
{% endhighlight %}

This seems pretty straight forward right? We could probably get away with a pretty simple addition...

{% highlight ruby %}
class LabLocations
  ...

  def fetch
    @results = []

    begin
      @results = legacy_locations.concat(future_locations)
    rescue => e
      Rails.logger.error(e)
    end

    @results
  end

  private

  def legacy_locations
    HTTParty.get('https://legacy-labs.com/locations', query: { zip_code: @zip_code }) rescue []
  end

  def future_locations
    HTTParty.get('https://future-labs.com/labLocations', query: { postalCode: @zip_code }) rescue []
  end
end
{% endhighlight %}

There is just a couple of problems here. First, location records returned by each provider have different attributes and some of the attribute values come in different formats. For example, `LegacyLab.hours` is a block of text with newline characters. `FutureLab.business_hours` returns back an array of hashes. To make matters even more complicated, the two labs share various lab locations.

{% highlight ruby %}
class LabLocations
  def initialize(zip_code)
    @zip_code = zip_code
  end

  def all
    legacy + future
  end

  def legacy
    HTTParty.get('https://legacy-labs.com/locations', query: { zip_code: @zip_code }) rescue []
  end


  def future
    HTTParty.get('https://future-labs.com/labLocations', query: { postalCode: @zip_code }) rescue []
  end
end
{% endhighlight %}

This isn't too bad. We now have an object that knows how to return the various location types or the abiltiy to combine them. However, we still don't have location objects that use the same language.

{% highlight ruby %}
class LabLocations
  def initialize(zip_code)
    @zip_code = zip_code
  end

  def all
    LegacyLabLocationAdapter.new(@zip_code).all + FutureLabLocationAdapter.new(@zip_code).all
  end
end

def LabLocationAdapter
  def initialize(zip_code)
    @zip_code = zip_code
  end

  def all
    raise "Not Yet Implemented"
  end
end

class LegacyLabLocationAdapter < LabLocationAdapter
  def all
    locations = HTTParty.get('https://legacy-labs.com/locations', query: { zip_code: @zip_code }) rescue []
    standardize_locations(locations)
  end

  private

  def standardize_locations(locations)
    locations.map do |location|
      LabLocation.new.tap do |lab_location|
        lab_location.hours = location.hours
        lab_location.address = location.address
      end
    end
  end
end


class FutureLabLocationAdapter < LabLocationAdapter
  def all
    locations = HTTParty.get('https://future-labs.com/labLocations', query: { postalCode: @zip_code }) rescue []
    standardize_locations(locations)
  end

  private

  def standardize_locations(locations)
    locations.map do |location|
      LabLocation.new.tap do |lab_location|
        lab_location.hours = hours_text_from_data(location.hours)
        lab_location.address = location.address
      end
    end
  end

  def hours_text_from_data(hours)
    text = ""

    hours.each do |day, hours_value|
      text << "#{day}: #{hours_value}\n"
    end

    text
  end
end
{% endhighlight %}

Let's go over this piece by piece. First we separate we the logic for fetching lab locations into two distinct objects. These objects are responsible for returning valid `LabLocation` objects from a given source. To make things a little cleaner we created a little `LabLocationAdapter` that defined a common interface for our adapter classes. This makes it easier to create other adapters for additional data sources while maintaining a consistent structure and method signature layout. What's the benefit of that? Let's take this one final step further.

{% highlight ruby %}
class LabLocations
  ADAPTERS = [
    LegacyLabLocationAdapter,
    FutureLabLocationAdapter,
    StarLabLocationAdapter
  ]

  def initialize(zip_code)
    @zip_code = zip_code
  end

  def all
    results = []
    ADAPTERS.each { |adapter| results << adapter.new(@zip_code).all }
    results
  end
end

class StarLabLocationAdapter < LabLocationAdapter
  def all
    locations = HTTParty.get('https://star-labs.com/locations', query: { postalCode: @zip_code }) rescue []
    standardize_locations(locations)
  end

  private

  def standardize_locations(locations)
    locations.map do |location|
      LabLocation.new.tap do |lab_location|
        lab_location.hours = hours
        lab_location.address = location.full_address
      end
    end
  end
end
{% endhighlight %}

Nice! Now we have a really clean way to add new providers with complete transparency to the code that calls out to `LabLocations.all`. This means we can maintain our usual interface, but the details under the hood have changed to support our new requirements.

We are almost there. The last requirement is that when a customer receives results from a lab, the application needs to parse and store those for the customers to access as they always have. For brevity sake lets just create the object structure and leave the implementation details for our imaginations.

First let's peak at the current implementation.

{% highlight ruby %}
class CreateResultsForOrder
  def initialize(order)
    @order = order
  end

  def call
    results = HTTParty.get("https://lagacy-labs.com/orders/#{@order.provider_id}/results") rescue []
    results.each do |result|
      order_result = OrderResult.from_legacy_result(result)
      order_result.order = @order
      order_result.save!
    end
    true
  end
end
{% endhighlight %}

If we take our adapter approach and apply it here, we might see something like this:

{% highlight ruby %}
class CreateResultsForOrder
  def initialize(order)
    @order = order
  end

  def call
    adapter = adapter_for_order.new(@order)
    adapter.call
  end

  private

  def adapter_for_order
    case @order.lab_provider
    when :legacy_labs then LegacyLabsOrderResultsAdapter
    when :future_labs then FutureLabsOrderResultsAdapter
    when :star_labs then StarLabsOrderResultsAdapter
    else
      raise "No adapter specified for #{@order.lab_provider}"
    end
  end
end

class OrderResultsAdapter
  def initialize(order)
    @order = order
  end

  def call
    raise "Not yet implemented!"
  end
end

class LegacyLabsOrderResultsAdapter < OrderResultsAdapter
  def call
    # implementation details here
  end
end

class FutureLabsOrderResultsAdapter < OrderResultsAdapter
  def call
    # implementation details here
  end
end

class StarLabsOrderResultsAdapter < OrderResultsAdapter
  def call
    # implementation details here
  end
end
{% endhighlight %}

Using the adpater pattern again, we can now fetch results from all three of our lab providers, transform them to `OrderResult` objects, and persist them to the database. All while maintaining the public interface of our original `CreateResultsForOrder` class.

The adapter pattern is a wonderful tool for abstracting away varying interfaces to a particular problem. Whether it is writing to something that acts like a file, persisting data in database through various engines, or even pulling Geolocation data out of various sources. I hope you all have enjoyed the topic today and I wish you all happy coding.
