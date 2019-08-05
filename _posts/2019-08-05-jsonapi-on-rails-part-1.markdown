---
layout: post
title:  "JSON:API On Rails, Part 1 of 2"
date:   2019-08-05
categories: software,json,api,rails,ruby
---

As promised, this week we are going to build a JSON API with Ruby on Rails. Before we get started though let's make sure everyone is setup and knows what we are building. First the tech requirements:

* Ruby 2.6.3
* Rails 6.0.0.rc2
* PostgreSQL
* Insomnia (for local API requests on OSX)

## What We Are Building
We are going to build a light rewrite of the Basecamp 3 API in our lovely new [JSON:API](https://jsonapi.org/format/1.1/) format. For the sake of time and simplicity we will be skipping the authentication workflow and instead will dive right into creating our data graph.

We will be referencing the guides and docs from [Graphiti](https://www.graphiti.dev/quickstart) to build our project. In addition we are going to try to build this thing following BDD principles with RSpec.

## Create The New Application

To create a new Rails API we are going to use the Graphiti template to generate the new project. We are going to call this app "Less Projects". In the command below you will notice the `-T` flag which tells Rails to skip Test::Unit. Since we are using RSpec there is no need for the second testing library.

{% highlight bash %}
# Generate the project
rails new less_projects -d postgresql -T --api -m https://www.graphiti.dev/template

# Press enter when prompted to change Graphiti's defaults
# This will generate our routes under `/api/v1`

# Switch to the app directory
cd less_projects

# create the database
rails db:create
{% endhighlight %}

## Projects

Projects will serve as our top level object. They are a container for all other resources.

### Creating Projects

Graphiti and Rails provides us with some wonderful "generators" to get started.

{% highlight bash %}
# first generate the model
rails g model project name:string description:text purpose:string status:string

# migrate the database (this adds the projects table with our fields to the DB)
rails db:migrate

# Finally, generate our first Graphiti resource
rails g graphiti:resource Project name:string description:string purpose:string status:string
{% endhighlight %}

Let's step through these generated files quick. Graphiti just handled a lot of boilerplate for us. If you look at the ProjectsController (`app/controllers/projects_controller.rb`) you will see that all 5 of the basic CRUD actions are covered. You might also notice references to `ProjectResource`. This might be a little unfamiliar. It's a construct that Graphiti introduces to perform most of the API behaviors. It handles fetching, mutating, filtering, pagination, and all kinds of useful things.

{% highlight ruby %}
class ProjectResource < ApplicationResource
  attribute :name, :string
  attribute :description, :string
  attribute :purpose, :string
  attribute :status, :string
end
{% endhighlight %}

Next let's take a look at the ProjectResource. Again these can do a lot of things, but let's start with our bare bones. One of the key things a resource does is that it serializes (transforms) the Project model instances into JSON:API compliant JSON objects. In order to do that you must define what fields or "attributes" you want serialized. For our purposes these defaults (above) will work just fine.

Before we move into the tests let's put them in a more typical hierarchy. RSpec has a fairly default folder hierarchy and moving our project into that will match more of what you will see out there in the wild. In this case it clearly marks our api specs as "request specs".

{% highlight bash %}
mkdir spec/requests
mv spec/api/ spec/requests/api
{% endhighlight %}

### Listing Projects

The test we will start with is listing projects. We can see that Graphiti was very helpful and wrote our first test for us. That's pretty sweet! There are a few small things hidden from us. The first is `jsonapi_get`. This method is included by the Graphiti spec helper gem. It abstracts away some of the implementation details of the JSON:API spec, chiefly headers, to remove some test boilerplate and make tests easier to write.

{% highlight ruby %}
require 'rails_helper'

RSpec.describe "projects#index", type: :request do
  let(:params) { {} }

  subject(:make_request) do
    jsonapi_get "/api/v1/projects", params: params
  end

  describe 'basic fetch' do
    let!(:project1) { create(:project) }
    let!(:project2) { create(:project) }

    it 'works' do
      expect(ProjectResource).to receive(:all).and_call_original
      make_request
      expect(response.status).to eq(200), response.body
      expect(d.map(&:jsonapi_type).uniq).to match_array(['projects'])
      expect(d.map(&:id)).to match_array([project1.id, project2.id])
    end
  end
end
{% endhighlight %}

The other thing it abstracts is the parsed response from the request. It's hidden away in the `d` variable. This isn't exactly named well so let's use a different method that Graphiti supports and reads a little easier: "jsonapi_data".

{% highlight ruby %}
# spec/requests/api/v1/projects/index_spec.rb

# old code
# expect(d.map(&:jsonapi_type).uniq).to match_array(['projects'])
# expect(d.map(&:id)).to match_array([project1.id, project2.id])

# new code
expect(jsonapi_data.map(&:jsonapi_type).uniq).to match_array(['projects'])
expect(jsonapi_data.map(&:id)).to match_array([project1.id, project2.id])
{% endhighlight %}

Alright, now let's run our test and see if things work.

{% highlight bash %}
rspec spec/requests/api/v1/projects/index_spec.rb

# output
Finished in 0.22877 seconds (files took 2.76 seconds to load)
2 examples, 0 failures
{% endhighlight %}

Woot! Our test already passes. Let's take a moment to boot up the Insomnia app and take a look at our payloads. All we need to do is a little bit of setup first and start our server.

First let's modify our default factory so that we can identify projects a little easier.

{% highlight ruby %}
# spec/factories/projects.rb

FactoryBot.define do
  factory :project do
    sequence(:name) { |n| "Project #{n}" }
    description { "Something cool" }
    purpose { "To organize the project" }
    status { "active" }
  end
end
{% endhighlight %}

Next we setup some data for our API.

{% highlight bash %}
# boot up rails console
rails console

# once in console, seed a few projects
FactoryBot.create_list(:project, 3)

# exit out of console
exit

# and finally start our server
rails s
{% endhighlight %}

The server should now be running at http://localhost:3000. We can at last jump into Insomnia and see what our API returns for our first endpoint.

Step 1: Create a new Request

![new request](https://p67.tr3.n0.cdn.getcloudapp.com/items/QwueQGAJ/new_request.png)

Step 2: Setup the default headers and Url

JSON:API Spec and Graphiti requires us to specify the format of the data for the server (Content-Type) and the format of data we expect to be returned (Accept). Both values will be `application/vnd.api+json`.

![default headers](https://p67.tr3.n0.cdn.getcloudapp.com/items/YEu8vdZL/jsonapi_headers.png)

Step 3: Click "Send" and win!

Here is our data. You will notice there are 2 top level fields: "data" and "meta". Data is where all of our primary data will be contained. The "meta" object is for more miscallenous types of data such as counts of records, pagination, etc. You might also notice that each "attributes" object has all of the fields we specified in our `ProjectResource` class. Not to shabby.

![projects response json](https://p67.tr3.n0.cdn.getcloudapp.com/items/rRuEnLZl/projects_index_json.png)

### Fetching Single Projects

The next thing we want to make sure our clients can do is look up single projects.  Again, like good devs let's start with the test.

{% highlight ruby %}
# spec/requests/api/v1/projects/show_spec.rb
require 'rails_helper'

RSpec.describe "projects#show", type: :request do
  let(:params) { {} }

  subject(:make_request) do
    jsonapi_get "/api/v1/projects/#{project.id}", params: params
  end

  describe 'basic fetch' do
    let!(:project) { create(:project) }

    it 'works' do
      expect(ProjectResource).to receive(:find).and_call_original
      make_request
      expect(response.status).to eq(200)
      expect(jsonapi_data.jsonapi_type).to eq('projects')
      expect(jsonapi_data.id).to eq(project.id)
    end
  end
end
{% endhighlight %}

And now we run the test.

{% highlight bash %}
rspec spec/requests/api/v1/projects/show_spec.rb

# output
Finished in 0.1661 seconds (files took 2 seconds to load)
2 examples, 0 failures
{% endhighlight %}

Just like before we find out our work was done for us and the tests pass. And just like before let's go into Insomnia and add our request.

*Pro Tip:* Right click the previous request and click duplicate to start with a request with the required headers.

![fetch a project](https://p67.tr3.n0.cdn.getcloudapp.com/items/7Ku2xRoD/projects_show.png)

### Creating Projects

Now that we have fetching of projects in place, let's learn how to create new ones in our API. Like before, we will start with the test.

{% highlight ruby %}
# spec/requests/v1/api/projects/create_spec.rb

require 'rails_helper'

RSpec.describe "projects#create", type: :request do
  subject(:make_request) do
    jsonapi_post "/api/v1/projects", payload
  end

  describe 'basic create' do
    let(:params) { attributes_for(:project) }

    let(:payload) do
      {
        data: {
          type: 'projects',
          attributes: params
        }
      }
    end

    it 'works' do
      expect(ProjectResource).to receive(:build).and_call_original
      expect { make_request }.to change { Project.count }.by(1)
      expect(response.status).to eq(201)
    end
  end
end
{% endhighlight %}

The are a few key things to point out here:

* All payloads start with a `data` hash/object
* `type` is required (this is the pluralized name of the object)
* `attributes` is where all the non-relationship fields are contained

If you are not familiar with `attributes_for` don't worry. This is just a convenient way to extract random attributes for a given factory. Let's check this out in Insomnia so we can see a real world example.

Step 1: Create the request

Again we need to duplicate and rename the previous request. Also, let's make sure to switch from a GET request to a POST request. When creating new resources over the API, the POST method is what the JSON:API spec requires.

![create new request](https://p67.tr3.n0.cdn.getcloudapp.com/items/8LudZPm5/projects_create_request.png)

Step 2: Create the POST body and Change the URL

We should make sure to change the url to reference `/api/v1/projects`. Think of this as the "root" url for projects. Making a GET request to this endpoint fetches projects and making a POST request creates them. Also make sure to change the request content to "JSON".

You will also notice that much like our responses from the previous requests we are making use of the `data` field as the root element and passing our desired traits through the `attributes` hash. This is a pattern we will continue to re-use.

![update request data](https://p67.tr3.n0.cdn.getcloudapp.com/items/Z4uZzkbG/projects_create_json.png)

Step 3: Click "Send" and win!

![create project response](https://p67.tr3.n0.cdn.getcloudapp.com/items/eDuoLBBl/projects_create_response.png)

Boom, just like that we have our new project. But what if we add some validations to projects and our request is invalid? Great question! Let's make some quick alterations and see what happens. We start by writing a failing test.

{% highlight ruby %}
# spec/models/project_spec.rb

require 'rails_helper'

RSpec.describe Project, type: :model do
  describe 'validations' do
    let(:project) { described_class.new }

    it 'requires a name' do
      expect(project).not_to be_valid
      expect(project.errors["name"]).to include("can't be blank")
    end
  end
end
{% endhighlight %}

And then we write the code to make the tests pass.

{% highlight ruby %}
# app/models/project.rb

class Project < ApplicationRecord
  validates :name, presence: true
end
{% endhighlight %}

If we did everything properly then our tests should pass.

{% highlight bash %}
rspec spec/models/project_spec.rb

# output
Finished in 1.03 seconds (files took 0.84581 seconds to load)
2 examples, 0 failure
{% endhighlight %}

B E A utiful! Our test now passes. Let's try to create a new project except we won't give it a name. Since we just added the feature we know this should fail.

Here is what we we see if we try this in Insomnia:

![invalid project](https://p67.tr3.n0.cdn.getcloudapp.com/items/E0ulPnXv/projects_create_invalid_json.png)

From a client standpoint this is pretty nice. We are given a response that tells us what attribute are invalid and why.

### Updating a Project

Now that we can create projects, let's learn how to update existing ones in our API. Like before, we will start with the test.

{% highlight ruby %}
# spec/requests/v1/api/projects/update_spec.rb

require 'rails_helper'

RSpec.describe "projects#update", type: :request do
  subject(:make_request) do
    jsonapi_put "/api/v1/projects/#{project.id}", payload
  end

  describe 'basic update' do
    let!(:project) { create(:project) }

    let(:payload) do
      {
        data: {
          id: project.id.to_s,
          type: 'projects',
          attributes: {
            name: "new name"
          }
        }
      }
    end

    it 'updates the resource' do
      expect(ProjectResource).to receive(:find).and_call_original
      make_request
      expect(response.status).to eq(200), response.body
      expect(project.reload.name).to eq("new name")
    end
  end
end
{% endhighlight %}

Just like when we create a project, we pass our json with "data" as the top level attribute. Let's check this out in Insomnia so we can see a real world example.

This time we will combine all the steps into 1.

![update project request](https://p67.tr3.n0.cdn.getcloudapp.com/items/KouLQnq0/Update+Project.png)

And that's all it takes to update the project.

### Destroying a Project

The last operation that we need to support is destroying a project. Just like in all the other examples...we will start with a test.

{% highlight ruby %}
# spec/requests/v1/api/projects/destroy_spec.rb

require 'rails_helper'

RSpec.describe "projects#destroy", type: :request do
  subject(:make_request) do
    jsonapi_delete "/api/v1/projects/#{project.id}", {}
  end

  describe 'basic destroy' do
    let!(:project) { create(:project) }

    it 'updates the resource' do
      expect(ProjectResource).to receive(:find).and_call_original
      make_request
      expect(response.status).to eq(200)
      expect { project.reload }.to raise_error(ActiveRecord::RecordNotFound)
    end
  end
end
{% endhighlight %}

If we run our tests, everything passes. That's great, but what kind of response do we get from this request? Let's dive back into Insomnia and add another request; just like we have previously.

![Destroy project request](https://p67.tr3.n0.cdn.getcloudapp.com/items/yAuo4AYR/delete_project.png)

The most notable thing in this request is that our response is quite different from the other ones. In this case only meta data is returned.

These 5 actions: list, fetch, update, create, and destroy are the main operations we will perform on each resource. Now let's move onto some nested resources and see how Graphiti handles them.

## TodoLists and TodoItems

Each project can have one or many todo lists. Each of these lists have todo items. Let's use our generators again to create these resources.

{% highlight bash %}
# setup todo_lists
rails g model todo_list project:references name:string

rails db:migrate

rails g graphiti:resource todo_list name:string

# setup todo_list_items
rails g model todo_list_item todo_list:references content:string complete:boolean

rails db:migrate

rails g graphiti:resource todo_list_item content:string complete:boolean
{% endhighlight %}

And just like before let's move our tests under `spec/requests`.

{% highlight bash %}
mv spec/api/v1/todo_lists spec/requests/api/v1
mv spec/api/v1/todo_list_items spec/requests/api/v1
{% endhighlight %}

Now let's alter our models and add the various relationships and default validations.

{% highlight ruby %}
# app/models/project.rb

class Project < ApplicationRecord
  has_many :todo_lists
  has_many :todo_list_items, through: :todo_lists

  validates :name, presence: true
end
{% endhighlight %}

{% highlight ruby %}
# app/models/todo_list.rb

class TodoList < ApplicationRecord
  belongs_to :project
  has_many :todo_list_items

  validates :name, presence: true
end
{% endhighlight %}

{% highlight ruby %}
# app/models/todo_list_item.rb

class TodoListItem < ApplicationRecord
  belongs_to :todo_list

  validates :content, presence: true
end
{% endhighlight %}

Next up, let's alter our resources to add those same relations.

{% highlight ruby %}
# app/resources/project_resource.rb

class ProjectResource < ApplicationResource
  has_many :todo_lists
  has_many :todo_list_items

  attribute :name, :string
  attribute :description, :string
  attribute :purpose, :string
  attribute :status, :string
end
{% endhighlight %}

{% highlight ruby %}
# app/resources/todo_list_resource.rb

class TodoListResource < ApplicationResource
  belongs_to :project
  has_many :todo_list_items

  attribute :name, :string
end
{% endhighlight %}

{% highlight ruby %}
# app/resources/todo_list_item_resource.rb

class TodoListItemResource < ApplicationResource
  belongs_to :todo_list

  attribute :content, :string
  attribute :complete, :boolean
end
{% endhighlight %}

Let's also make some changes to the new factories so that our tests are a bit more stable and the data a bit more random.

{% highlight ruby %}
# spec/factories/todo_list_items.rb

FactoryBot.define do
  factory :todo_list_item do
    todo_list
    content { Faker::Lorem.words(number: 5).join('') }
    complete { false }
  end
end
{% endhighlight %}

{% highlight ruby %}
# spec/factories/todo_lists.rb

FactoryBot.define do
  factory :todo_list do
    project
    sequence(:name) { |n| "#{Faker::Beer.name} #{n}" }
  end
end
{% endhighlight %}

Alright, now that we have all of the foundations in place, let's move into our new `TodoList` resource.

### Listing TodoLists

In order to give Insomnia some data to display, seed some todo lists.

{% highlight bash %}
rails c

# in console
# seeds 3 todo lists for the last project we created
FactoryBot.create_list(:todo_list, 3, project: Project.last)
{% endhighlight %}

Then create and execute the new request in Insomnia

![list todo lists screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/kpuydzqB/todo_lists_index.png?v=ca822979dcbf3a9b534991761a0f54e4)

And of course we need to check our test.

{% highlight ruby %}
spec/requests/api/v1/todo_lists/index_spec.rb

require 'rails_helper'

RSpec.describe "todo_lists#index", type: :request do
  let(:params) { {} }

  subject(:make_request) do
    jsonapi_get "/api/v1/todo_lists", params: params
  end

  describe 'basic fetch' do
    let!(:todo_list1) { create(:todo_list) }
    let!(:todo_list2) { create(:todo_list) }

    it 'works' do
      expect(TodoListResource).to receive(:all).and_call_original
      make_request
      expect(response.status).to eq(200)
      expect(d.map(&:jsonapi_type).uniq).to match_array(['todo_lists'])
      expect(d.map(&:id)).to match_array([todo_list1.id, todo_list2.id])
    end
  end
end
{% endhighlight %}

If we run our test provided by Graphiti we see green!

{% highlight bash %}
Finished in 0.2598 seconds (files took 1.5 seconds to load)
2 examples, 0 failures
{% endhighlight %}

### Fetching TodoLists

Like before, let's create and execute the a new request in Insomnia to fetch a single `TodoList`.

![fetch todo list screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/7Ku2xwK7/todo_lists_show.png?v=94fe7d3e55c53bab288ea309dbc6b075)

And of course we need to check our test.

{% highlight ruby %}
spec/requests/api/v1/todo_lists/show_spec.rb

require 'rails_helper'

RSpec.describe "todo_lists#show", type: :request do
  let(:params) { {} }

  subject(:make_request) do
    jsonapi_get "/api/v1/todo_lists/#{todo_list.id}", params: params
  end

  describe 'basic fetch' do
    let!(:todo_list) { create(:todo_list) }

    it 'works' do
      expect(TodoListResource).to receive(:find).and_call_original
      make_request
      expect(response.status).to eq(200)
      expect(d.jsonapi_type).to eq('todo_lists')
      expect(d.id).to eq(todo_list.id)
    end
  end
end
{% endhighlight %}

If we run our test provided by Graphiti we see green!

{% highlight bash %}
Finished in 0.19348 seconds (files took 0.32548 seconds to load)
2 examples, 0 failures
{% endhighlight %}

### Creating TodoList

Let's create and execute the a new request in Insomnia to create a single `TodoList`.

![create todo list screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/xQu9vkBl/todo_lists_create.png?v=24cc20033092f4767faf4dbabb95a862)

And of course we need to write/fix our test that was autogenerated by Graphiti.

{% highlight ruby %}
spec/requests/api/v1/todo_lists/create_spec.rb

require 'rails_helper'

RSpec.describe "todo_lists#create", type: :request do
  let!(:project) { create(:project) }

  subject(:make_request) do
    jsonapi_post "/api/v1/todo_lists", payload
  end

  describe 'basic create' do
    let(:params) { attributes_for(:todo_list) }

    let(:payload) do
      {
        data: {
          type: 'todo_lists',
          attributes: params,
          relationships: {
            project: {
              data: {
                type: 'projects',
                id: project.id
              }
            }
          }
        }
      }
    end

    it 'works' do
      expect(TodoListResource).to receive(:build).and_call_original
      expect { make_request }.to change { TodoList.count }.by(1)
      expect(response.status).to eq(201)
    end
  end
end
{% endhighlight %}

If we run our test provided we see green!

{% highlight bash %}
Finished in 0.23687 seconds (files took 0.20839 seconds to load)
2 examples, 0 failures
{% endhighlight %}

### Updating TodoLists

Let's create and execute the a new request in Insomnia to update a `TodoList`.

![update a todo list screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/7Ku2xw7k/todo_lists_update.png?v=80d2784f35697c51afe245570b618cbc)

And of course we need to write/fix our test that was autogenerated by Graphiti.

{% highlight ruby %}
spec/requests/api/v1/todo_lists/update_spec.rb

require 'rails_helper'

RSpec.describe "todo_lists#update", type: :request do
  subject(:make_request) do
    jsonapi_put "/api/v1/todo_lists/#{todo_list.id}", payload
  end

  describe 'basic update' do
    let!(:todo_list) { create(:todo_list) }

    let(:payload) do
      {
        data: {
          id: todo_list.id.to_s,
          type: 'todo_lists',
          attributes: {
            name: "new name"
          }
        }
      }
    end

    it 'updates the resource' do
      expect(TodoListResource).to receive(:find).and_call_original
      make_request
      expect(response.status).to eq(200)

      expect(todo_list.reload.name).to eq("new name")
    end
  end
end

{% endhighlight %}

If we run our test provided we see green!

{% highlight bash %}
Finished in 0.21061 seconds (files took 0.22622 seconds to load)
2 examples, 0 failures
{% endhighlight %}

### Destroying TodoLists

Let's create and execute the a new request in Insomnia to destroy a `TodoList`.

![destroy a todo list screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/wbuJedjP/todo_lists_destroy.png?v=b111453f68ebf2256f2e03ecebebe62a)

And of course we need to write/fix our test that was autogenerated by Graphiti.

{% highlight ruby %}
spec/requests/api/v1/todo_lists/destroy_spec.rb

require 'rails_helper'

RSpec.describe "todo_lists#destroy", type: :request do
  subject(:make_request) do
    jsonapi_delete "/api/v1/todo_lists/#{todo_list.id}", {}
  end

  describe 'basic destroy' do
    let!(:todo_list) { create(:todo_list) }

    it 'updates the resource' do
      expect(TodoListResource).to receive(:find).and_call_original
      make_request
      expect(response.status).to eq(200)
      expect { todo_list.reload }.to raise_error(ActiveRecord::RecordNotFound)
      expect(json).to eq('meta' => {})
    end
  end
end
{% endhighlight %}

If we run our test provided we see green!

{% highlight bash %}
Finished in 0.22919 seconds (files took 0.20982 seconds to load)
2 examples, 0 failures
{% endhighlight %}

Whew that's two of the three resources covered. Time to rip through the `TodoListItem` and tease some of the content from part 2 of this article.

### Listing TodoListItems

In order to give Insomnia some data to display, seed some todo list items.

{% highlight bash %}
rails c

# in console
FactoryBot.create_list(:todo_list_item, 5, todo_list: TodoList.last)
{% endhighlight %}

Then create and execute the new request in Insomnia

![list todo lists screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/GGuL09XB/todo_list_items_index.png?v=9e2270918b9a22b455c5d8ec6a5ff14f)

And of course we need to check our test.

{% highlight ruby %}
spec/requests/api/v1/todo_list_items/index_spec.rb

require 'rails_helper'

RSpec.describe "todo_list_items#index", type: :request do
  let(:params) { {} }

  subject(:make_request) do
    jsonapi_get "/api/v1/todo_list_items", params: params
  end

  describe 'basic fetch' do
    let!(:todo_list_item1) { create(:todo_list_item) }
    let!(:todo_list_item2) { create(:todo_list_item) }

    it 'works' do
      expect(TodoListItemResource).to receive(:all).and_call_original
      make_request
      expect(response.status).to eq(200), response.body
      expect(d.map(&:jsonapi_type).uniq).to match_array(['todo_list_items'])
      expect(d.map(&:id)).to match_array([todo_list_item1.id, todo_list_item2.id])
    end
  end
end
{% endhighlight %}

If we run our test provided by Graphiti we see green!

{% highlight bash %}
rspec spec/requests/api/v1/todo_list_items/index_spec.rb

Finished in 0.27564 seconds (files took 0.2522 seconds to load)
2 examples, 0 failures
{% endhighlight %}

### Fetching TodoListItems

Like before, let's create and execute the a new request in Insomnia to fetch a single `TodoListItem`.

![fetch a todo list item screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/12uOEmqZ/todo_list_items_show.png?v=e141bcea712aed5930749f3effe12480)

And of course we need to check our test.

{% highlight ruby %}
spec/requests/api/v1/todo_list_items/show_spec.rb

require 'rails_helper'

RSpec.describe "todo_list_items#show", type: :request do
  let(:params) { {} }

  subject(:make_request) do
    jsonapi_get "/api/v1/todo_list_items/#{todo_list_item.id}", params: params
  end

  describe 'basic fetch' do
    let!(:todo_list_item) { create(:todo_list_item) }

    it 'works' do
      expect(TodoListItemResource).to receive(:find).and_call_original
      make_request
      expect(response.status).to eq(200)
      expect(d.jsonapi_type).to eq('todo_list_items')
      expect(d.id).to eq(todo_list_item.id)
    end
  end
end
{% endhighlight %}

If we run our test provided by Graphiti we see green!

{% highlight bash %}
rspec spec/requests/api/v1/todo_list_items/show_spec.rb

Finished in 0.18498 seconds (files took 0.23766 seconds to load)
2 examples, 0 failures
{% endhighlight %}

### Creating TodoListItems

Let's create and execute the a new request in Insomnia to create a single `TodoListItem`.

![create todo list item screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/WnuwA9dr/todo_list_items_create.png?v=5041ccbf3dcde3cc8c098df9ac45bdbe)

And of course we need to write/fix our test that was autogenerated by Graphiti.

{% highlight ruby %}
spec/requests/api/v1/todo_list_items/create_spec.rb

require 'rails_helper'

RSpec.describe "todo_list_items#create", type: :request do
  subject(:make_request) do
    jsonapi_post "/api/v1/todo_list_items", payload
  end

  describe 'basic create' do
    let(:params) { attributes_for(:todo_list_item) }
    let!(:todo_list) { create(:todo_list) }

    let(:payload) do
      {
        data: {
          type: 'todo_list_items',
          attributes: params,
          relationships: {
            todo_list: {
              data: {
                type: 'todo_lists',
                id: todo_list.id
              }
            }
          }
        }
      }
    end

    it 'works' do
      expect(TodoListItemResource).to receive(:build).and_call_original
      expect { make_request }.to change { TodoListItem.count }.by(1)
      expect(response.status).to eq(201)
    end
  end
end
{% endhighlight %}

If we run our test provided we see green!

{% highlight bash %}
rspec spec/requests/api/v1/todo_list_items/create_spec.rb

Finished in 0.23428 seconds (files took 0.25014 seconds to load)
2 examples, 0 failures
{% endhighlight %}

### Updating TodoListItems

Let's create and execute the a new request in Insomnia to update a `TodoListItem`.

![update a todo list screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/P8uWlzld/todo_list_items_update.png?v=e86a6e125f97b539f3e306b47786db60)

And of course we need to write/fix our test that was autogenerated by Graphiti.

{% highlight ruby %}
spec/requests/api/v1/todo_list_items/update_spec.rb

require 'rails_helper'

RSpec.describe "todo_list_items#update", type: :request do
  subject(:make_request) do
    jsonapi_put "/api/v1/todo_list_items/#{todo_list_item.id}", payload
  end

  describe 'basic update' do
    let!(:todo_list_item) { create(:todo_list_item) }

    let(:payload) do
      {
        data: {
          id: todo_list_item.id.to_s,
          type: 'todo_list_items',
          attributes: {
            content: "break the dishes!"
          }
        }
      }
    end

    it 'updates the resource' do
      expect(TodoListItemResource).to receive(:find).and_call_original
      make_request
      expect(response.status).to eq(200)
      expect(todo_list_item.reload.content).to eq("break the dishes!")
    end
  end
end
{% endhighlight %}

If we run our test provided we see green!

{% highlight bash %}
rspec spec/requests/api/v1/todo_list_items/update_spec.rb

Finished in 0.30056 seconds (files took 0.25718 seconds to load)
2 examples, 0 failures
{% endhighlight %}

### Destroying TodoListItems

Let's create and execute the a new request in Insomnia to destroy a `TodoListItem`.

![destroy a todo list item screenshot](https://p67.tr3.n0.cdn.getcloudapp.com/items/d5uzbkep/todo_list_items_destroy.png?v=02571d2e222b9a11957652d802a46fb7)

And of course we need to write/fix our test that was autogenerated by Graphiti.

{% highlight ruby %}
spec/requests/api/v1/todo_list_items/destroy_spec.rb

require 'rails_helper'

RSpec.describe "todo_list_items#destroy", type: :request do
  subject(:make_request) do
    jsonapi_delete "/api/v1/todo_list_items/#{todo_list_item.id}", {}
  end

  describe 'basic destroy' do
    let!(:todo_list_item) { create(:todo_list_item) }

    it 'destroys the resource' do
      expect(TodoListItemResource).to receive(:find).and_call_original
      expect { make_request}.to change { TodoListItem.count }.by(-1)
      expect(response.status).to eq(200)
      expect { todo_list_item.reload }.to raise_error(ActiveRecord::RecordNotFound)
      expect(json).to eq('meta' => {})
    end
  end
end
{% endhighlight %}

If we run our test provided we see green!

{% highlight bash %}
rspec spec/requests/api/v1/todo_list_items/destroy_spec.rb

Finished in 0.38517 seconds (files took 0.56542 seconds to load)
2 examples, 0 failures
{% endhighlight %}

That covers all of our resources! Thanks for hanging in there with me while we stepped through this together. If you fast-forwarded, scrolled through, or encountered issues you can view the full repo [here](https://github.com/caseyprovost/less_projects/commit/7b9189ffc48001507f67fc7eefcd25726a44dcd7) for Part 1.

I cannot wait to get into part 2 and show you all where JSON:API and Graphiti really shine. I'll be covering some slightly more advanced topics such as side-posting, side-loading, filtering, sorting, and pagination.
