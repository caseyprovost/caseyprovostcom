---
layout: post
title:  "Query Objects and RSpec"
date:   2019-07-02
categories: ruby,design-patterns,testing
---

Today we will be continuing our conversation from the previous article: [Query Objects](https://caseyprovost.com/2019/7-days-of-design-patterns-part-5/). We will be using RSpec to unit test this new type of class. If you are not familiar with RSpec, they have some really [phenominal docs](https://relishapp.com/rspec).

Well let's dive right in! First, Let's capture our code from the previous episode (with some minor corrections).

{% highlight ruby %}
class IncomeReportQuery
  attr_reader :relation
  attr_reader :filters

  def initialize(relation:, filters: {})
    @relation = relation
    @filters = filters
  end

  def resolve
    scope = relation.have_income
    return scope.distinct if filters.empty?

    apply_filter(scope, :start_date)
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
    scope.where('created_at >= ?', value.to_datetime.beginning_of_day)
  end

  def filter_by_end_date(scope, value)
    scope.where('created_at <= ?', value.to_datetime.end_of_day)
  end

  def filter_by_user_status(scope, value)
    scope.joins(:user).where(users: { status: value })
  end

  def filter_by_amount(scope, value)
    scope.where('amount >= ?', value)
  end

  def filter_by_user_id(scope, value)
    scope.joins(:user).where(users: { id: value })
  end
end

{% endhighlight %}

Now let's wave our magic test wand and see what happens...WOOSH!

{% highlight ruby %}
require 'rails_helper'

RSpec.describe IncomeReportQuery do
  let!(:user) { create(:user) }

  describe '#resolve' do
    let(:result) { query.resolve }

    context 'filtering by nothing' do
      let(:query) { described_class.new(relation: Purchase.all, filters: {}) }
      let!(:purchase_1) { create(:purchase, :with_user) }
      let!(:purchase_2) { create(:purchase, :with_user, :income) }
      let!(:purchase_3) { create(:purchase, :with_user, :income) }

      it 'returns the expected purchases' do
        expect(result).to match_array([purchase_2, purchase_3])
      end
    end

    context 'filtering by start date' do
      let(:query) { described_class.new(relation: Purchase.all, filters: filters) }
      let(:filters) { { start_date: '2019-06-01' } }
      let!(:purchase_1) { create(:purchase, :with_user, :income, created_at: '2019-05-31 23:59:59') }
      let!(:purchase_2) { create(:purchase, :with_user, :income, created_at: '2019-06-01 00:00:00') }
      let!(:purchase_3) { create(:purchase, :with_user, :income, created_at: '2019-06-01 23:59:59') }

      it 'returns the expected purchases' do
        expect(result).to match_array([purchase_2, purchase_3])
      end
    end

    context 'filtering by end date' do
      let(:query) { described_class.new(relation: Purchase.all, filters: filters) }
      let(:filters) { { end_date: '2019-05-31' } }
      let!(:purchase_1) { create(:purchase, :with_user, :income, created_at: '2019-05-31 23:59:59') }
      let!(:purchase_2) { create(:purchase, :with_user, :income, created_at: '2019-06-01 00:00:00') }
      let!(:purchase_3) { create(:purchase, :with_user, :income, created_at: '2019-06-01 23:59:59') }

      it 'returns the expected purchases' do
        expect(result).to match_array([purchase_1])
      end
    end

    context 'filtering by start date and end date' do
      let(:query) { described_class.new(relation: Purchase.all, filters: filters) }
      let(:filters) { { start_date: '2019-05-31', end_date: '2019-06-01' } }
      let!(:purchase_1) { create(:purchase, :with_user, :income, created_at: '2019-05-31 23:59:59') }
      let!(:purchase_2) { create(:purchase, :with_user, :income, created_at: '2019-06-01 00:00:00') }
      let!(:purchase_3) { create(:purchase, :with_user, :income, created_at: '2019-06-01 23:59:59') }
      let!(:purchase_4) { create(:purchase, :with_user, :income, created_at: '2019-06-02 00:00:00') }

      it 'returns the expected purchases' do
        expect(result).to match_array([purchase_1, purchase_2, purchase_3])
      end
    end

    context 'filtering by user status' do
      let(:query) { described_class.new(relation: Purchase.all, filters: filters) }
      let(:filters) { { user_status: 'inactive' } }
      let!(:purchase_1) { create(:purchase, :with_user, :income) }
      let!(:purchase_2) { create(:purchase, :with_user, :income) }
      let!(:purchase_3) { create(:purchase, :income, user: create(:user, :inactive)) }

      it 'returns the expected purchases' do
        expect(result).to match_array([purchase_3])
      end
    end

    context 'filtering by user id' do
      let(:query) { described_class.new(relation: Purchase.all, filters: filters) }
      let(:filters) { { user_id: user.id } }
      let!(:purchase_1) { create(:purchase, :income, :with_user) }
      let!(:purchase_2) { create(:purchase, :income, user: user) }
      let!(:purchase_3) { create(:purchase, :income, user: user) }

      it 'returns the expected purchases' do
        expect(result).to match_array([purchase_2, purchase_3])
      end
    end

    context 'filtering by minimum amount' do
      let(:query) { described_class.new(relation: Purchase.all, filters: filters) }
      let(:filters) { { amount: '100.00' } }
      let!(:purchase_1) { create(:purchase, :with_user, :income, amount: '99.99') }
      let!(:purchase_2) { create(:purchase, :with_user, :income, amount: '100.56') }
      let!(:purchase_3) { create(:purchase, :with_user, :income, amount: '101.23') }

      it 'returns the expected purchases' do
        expect(result).to match_array([purchase_2, purchase_3])
      end
    end
  end
end
{% endhighlight %}

Wow, well that was rather unexpected, but we do have our tests here for us. Now I'm not gonna lie to you. There is a lot of little details like factory bot usage, scope definition, and time-sensitive-testing that I'm just going to ignore so we can talk about the important pieces. Without further ado, let's jump in.

First thing you might notice is that this is a rather long test file. The truth is that this query object covers six different combinations of queries. This means we need to create Purchase objects to fullfill the conditions of each of our scenarios. We also need to test different instances of the query object that contain our six different filters. This is why you see "let" all over the place. This allows us to setup our assertions.

The second thing you might notice is that there is zero mocking or stubbing here. You might say something like "but these tests are coupled to the DB so these tests will be slow". Again, you would be right. There are a couple problems with mocking here. First, we would need to create a mock that behaves like an activerecord scope. Second, we would need to to assert the correct methods are called with the right SQL/arguments. This is pretty complex to setup and in my opinion, takes away from the readability of the tests. I would also argue that a good unit test here would be to run "query.resolve" with various inputs and only test the output. You can read more about unit testing from men smarter than me like [martin fowler](https://www.martinfowler.com/bliki/UnitTest.html). At the end of the day we want to make sure that given a collection of Purchase objects...this query returns the correct results given a set of filters.

If any of the techniques or code in this example is unclear please drop a comment and I will respond. The project this code lives in is available on [Github](https://github.com/caseyprovost/query-object-example).

As always, thank you for taking the time to read my ramblings. You truly are the best!
