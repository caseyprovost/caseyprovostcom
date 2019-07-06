---
layout: post
title:  "Seven Days of Design Patterns | Part 6 - Presenters"
date:   2019-07-06
categories: ruby,design-patterns
---

Why hello there! Are you researching design patterns applied to ruby? Are you interested in learning the value of Presenters? Well then, you have come to the right place my friend. Presenters, in my opinion, are most useful when displaying information to users in your app's User Interface (UI). Let's check out the following Rails view code as a place rife with opportunity.

{% highlight html %}
<table data-collection="orders">
  <thead>
    <tr>
      <th>Status</th>
      <th>Order Number</th>
      <th>Customer</th>
      <th>Item Count</th>
      <th>Total</th>
    </tr>
  </thead>
  <tbody>
    <% @orders.each do |order| %>
      <tr>
        <td>
          <span class="tag <%= order_status_class(order) %>">
            <%= (order.order_statuses || []).last %>
          </span>
        </td>
        <td><%= order.order_number %></td>
        <td><%= order.customer&.name %></td>
        <td><%= order.line_items.count %></td>
        <td><%= Monetize.parse(order.line_items.sum(:amount)).format %></td>
      </tr>
    <% end %>
  </tbody>
</table>
{% endhighlight %}

While this isn't 100% awful..it is a tad messy and there is logic strewn about here and likely in the controller and even in a helper file. This means that you as a developer have to navigate around multiple file to understand how to render an order row in this table. That is not ideal. Let's create a presenter and see if we can clean things up.

{% highlight ruby %}
module OrderReport
  class OrderPresenter
    def initialize(order)
      @order = order
    end

    def status
      @status ||= @order.order_statuses.last&.name
    end

    def status_class
      case status
        when 'pending' then 'warning'
        when 'complete' then 'success'
        when 'refunded' then 'error'
        else
        'default'
      end
    end

    def order_number
      @order.order_number
    end

    def customer_name
      @order.customer&.name
    end

    def item_count
      line_items.count
    end

    def formatted_total
      total.format
    end

    private

    def total
      @total ||= Monetize.parse(line_items.sum(:amount))
    end

    def line_items
      @line_items ||= @order.line_items
    end
  end
end
{% endhighlight %}

And there it is! Our beautiful little presenter object. As you can see, we abstracted away quite a lot of logic. This is really great for a few reasons. The first is for testing. Can you imagine writing integration/system tests to assert on the various states of this view? You'd be writing 4 minutes of tests just to do that! Moving the logic to a presenter allows you to unit test all of those conditions on the new presenter object, which is much easier to reason about and much faster to test. The second and last reason I think this presenter is awesome is because of how flexible it is. If you wanted to change how currency is formatted or update the css classes for your tags...there is just one file and a couple of methods to change.

Ok, ok..enough chatting, let's use our new presenter to clean up that view!

{% highlight html %}
<table data-collection="orders">
  <thead>
    <tr>
      <th>Status</th>
      <th>Order Number</th>
      <th>Customer</th>
      <th>Item Count</th>
      <th>Total</th>
    </tr>
  </thead>
  <tbody>
    <% @orders.map { |order| OrderReport::OrderPresenter.new(order) }.each do |presenter| %>
      <tr>
        <td>
          <span class="tag <%= presenter.status_class %>">
            <%= presenter.status %>
          </span>
        </td>
        <td><%= presenter.order_number %></td>
        <td><%= presenter.customer_name %></td>
        <td><%= presenter.item_count %></td>
        <td><%= presenter.formatted_total %></td>
      </tr>
    <% end %>
  </tbody>
</table>
{% endhighlight %}

That is a pretty sexy view if you ask me. We could of course pull *@orders* out and make it an array of presenter objects in our controller..but this transformation in the view works for our purpose: demonstration. As you can see, the rending of this view has become increasingly boring.

At the end of the day this is what applying design patterns is all about. They are used to enhance development and maintanability of code through simplification. After all boring code is often the easiest to work with.

Thank you so much for sticking around. Next week, we will be talking about Proxy objects. If you are interested please come back and read some more. Also, don't be afraid to hop in the comments below and tell me what you thought of this article or point out those pesky mistakes that I can fix.
