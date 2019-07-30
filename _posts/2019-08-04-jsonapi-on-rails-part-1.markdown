---
layout: post
title:  "JSON:API On Rails, Part 1 of 2"
date:   2019-08-04
categories: software,json,api,rails,ruby
---

Today I want a tackle a rather interesting debate. I'm sure there are valid use cases for both of these technologies...however after playing with both pretty extensively I wanted to share my findings. Let's talk about you for a second though. You probably found this article because you are looking to build an API. Maybe it's only an API, or maybe it's an API for a front-end application. Maybe you love GraphQL or maybe like me you are wary of it and are looking for a simpler alternative. Spoiler alert: JSON:API is a simpler and likely superior solution. Don't worry though you are about to find out why.

Let's build two straight forward APIs on Rails. This API is going to be for ACME Ocean Freight. They want to be to share and modify information about their cargo. The data model hierarchy is Ship/Container/Item.

To kick it off, we will be using Graphiti to build our JSON:API. The team at Graphiti has done some amazing work to give us a lot of spec-compliant functionality out of the box. Things like simple CRUD, filtering, sorting, pagination, and more.

```bash
# https://www.graphiti.dev/quickstart
rails new shipper -T -d=postgresql --api -m https://www.graphiti.dev/template
```

This command will setup the basic "frame" of our API, designates PostgreSQL as the DB, tells Rails to skip TestUnit, and finally only includes the components needed for an API.

Next we need to generate our data models. Let's start with ships.

```bash
rails g model ship name:string designation:string container_limit:integer
```

Most of what this generates is pretty solid, but we will want to tweak out migration and factory to be a little more sensible.

```ruby
# db/migrate/20190729172821_create_ships.rb

class CreateShips < ActiveRecord::Migration[6.0]
  def change
    create_table :ships do |t|
      t.string :name, null: false
      t.string :designation, null: false
      t.integer :container_limit, null: false, default: 1

      t.timestamps
    end
  end
end
```

```ruby
# spec/factories/ships.rb

FactoryBot.define do
  factory :ship do
    sequence(:name) { |n| "USS #{Faker::Movies::HarryPotter.spell}-#{n}" }
    designation { Faker::Number.number(4) }
    container_limit { Faker::Number.number(2) }
  end
end
```

Now we can create our new table and generate our first resource.

```bash
# create the table
rails db:migrate
```

```
# create our first API resource
rails g graphiti:resource Ship name:string designation:string container_limit:integer
```

This is pretty great! Graphiti went ahead and created some initial specs for us, and installed the proper route and controller code. Since we are disciplined engineers who write tests first, let's go ahead and run the ones we have.

```bash
Finished in 0.30403 seconds (files took 2.65 seconds to load)
14 examples, 1 failure, 3 pending
```

Oh man! It looks like there is a small bug in the generated code. Let' fix the affending file

```ruby
# spec/api/v1/ships/destroy_spec.rb

## Before
jsonapi_delete "/api/v1/ships/#{ship.id}"

## After
jsonapi_delete "/api/v1/ships/#{ship.id}", {}
```

We can verify this fixes it by running our tests again.

```bash
Finished in 0.22076 seconds (files took 0.84559 seconds to load)
14 examples, 0 failures, 3 pending
```

Now while we have seen some cool features of Graphiti...that's not really why you are here right? You want to see why this approach is better than jumping into the GraphQL pool. Let's add our child resources and a couple of filters to demonstrate how easy building this API truly is.

First our containers.

```bash
rails g model container number:string ship:references
```

Then our items.

```bash
rails g model item name:string identifier:string weight:decimal width:integer height:integer length:integer container:references
```

And finally, we creates the tables and run the migrations.

```bash
rails db:migrate
```

Now that all our tables, models, and factories are in place let's generate the resources.

```bash
# containers
rails g graphiti:resource container number:string

# items
rails g graphiti:resource item name:string identifier:string weight:big_decimal width:integer height:integer length:integer
```

If we run our tests again (after applying our "jsonapi_delete" fix from earlier) we will see that the world has broken a little.

```bash
Finished in 0.42436 seconds (files took 0.85432 seconds to load)
40 examples, 11 failures, 9 pending
```

Mostly this is because we need to open up our resources and models and specify the proper relationships.

```ruby
# app/models/container.rb

class Container < ApplicationRecord
  has_many :items
end
```

```ruby
# app/resources/container_resource.rb

class ContainerResource < ApplicationResource
  has_many :items

  attribute :number, :string
end
```

```ruby
# app/models/ship.rb

class Ship < ApplicationRecord
  has_many :containers
end
```

```ruby
# app/resources/ship_resource.rb

class ShipResource < ApplicationResource
  has_many :containers

  attribute :name, :string
  attribute :designation, :string
  attribute :container_limit, :integer
end
```

```ruby
# app/models/item.rb

class Item < ApplicationRecord
  belongs_to :container
end
```

```ruby
# app/resources/item_resource.rb

class ItemResource < ApplicationResource
  belongs_to :container

  attribute :name, :string
  attribute :identifier, :string
  attribute :weight, :big_decimal
  attribute :width, :integer
  attribute :height, :integer
  attribute :length, :integer
end
```

