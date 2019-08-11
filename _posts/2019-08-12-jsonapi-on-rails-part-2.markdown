---
layout: post
title:  "JSON:API On Rails (Part 2 of 2)"
date:   2019-08-11
categories: software,json,api,rails,ruby
---

Today we're going to dive right back into building our app. If you need a refresher, check out [part 1](https://caseyprovost.com/2019/jsonapi-on-rails-part-1/). We left off with the ability to CRUD Projects, TodoLists, and Todos. This is the common "boilerplate" that is often the foundation of APIs. Where APIs get more interesting though is in how they more complex topics such as searching, pagination, and nested writes. JSON:API has some wonderful answers to these things.

## Filtering

Filtering is one of the most common problems when building out your APIs. There are lots of ways to do this, but let's see how to do this with Graphiti. Let's add teh ability to filter projects by status. As usual, we begin with our unit test.

```ruby
context "by status" do
  before do
    project1.update!(status: 'active')
    project2.update!(status: 'archived')
    params[:filter] = { status: { eq: 'active' } }
  end

  it "returns the expected projects" do
    render
    expect(jsonapi_data.map(&:id)).to eq([project1.id])
  end
end
```

And if we run our tests we should see this fail.

```bash
spring rspec spec/resources/project/reads_spec.rb
Running via Spring preloader in process 81715
......

Finished in 0.21146 seconds (files took 0.2209 seconds to load)
6 examples, 0 failure
```

Wait a minute...why is our test passing? It's because Graphiti give us filtering for ALL of our defined attributes out of the box! This is pretty darn sweet. Something to call out here though is the structure of our filter. We start with an attribute name and then define the type of filter and pass the value. In this case  `status` is the attribute, `eq` is the type, and `active` is our value. What if we needed to extend this though so we could filter by multiple statuses? Let's add a test.

```ruby
context "by multiple status" do
  before do
    project1.update!(status: 'active')
    project2.update!(status: 'archived')
    project3.touch
    params[:filter] = { status: { eq: 'active,archived' } }
  end

  let!(:project3) { create(:project, status: "junk") }

  it "returns the expected projects" do
    render
    expect(jsonapi_data.map(&:id)).to eq([project1.id, project2.id])
  end
end
```

Then we run our tests.

```bash
spring rspec spec/resources/project/reads_spec.rb
Running via Spring preloader in process 81842
.......

Finished in 0.20526 seconds (files took 0.21959 seconds to load)
7 examples, 0 failures
```

Man! Looks like this was handled for us already too. Graphiti gives you a lot of power. I won't go into all the details, but you can read more [here](https://www.graphiti.dev/guides/concepts/resources#filter-options). What if we want a more complex filter though? Such as a filter that only returns projects that have a todo list? As usual, let's start off with a test.

```ruby
context "having at least 1 todo list" do
  before do
    todo_list.touch
    params[:filter] = { has_todo_lists: true }
  end

  let!(:todo_list) { create(:todo_list, project: project1) }

  it "returns the expected projects" do
    render
    expect(jsonapi_data.map(&:id)).to eq([project1.id])
  end
end
```

Let's run it.

```bash
spring rspec spec/resources/project/reads_spec.rb
Running via Spring preloader in process 87453
.....F..

Failures:

  1) ProjectResource filtering having at least 1 todo list returns the expected projects
     Failure/Error: render

     Graphiti::Errors::UnknownAttribute:
       ProjectResource: Tried to filter on attribute :has_todo_lists, but could not find an attribute with that name.
```

Uh oh, looks like we have a problem. That's ok though because we didn't expect a custom filter to just work "out of the box". Let's fix this.

```ruby
# app/resources/project_resource.rb

filter :has_todo_lists, :boolean do
    eq do |scope, value|
      if value
        scope.joins(:todo_lists)
      else
        scope.left_joins(:todo_lists).
          where("todo_lists.id is null")
      end
    end
  end
```

And we run our tests...

```bash
spring rspec spec/resources/project/reads_spec.rb
Running via Spring preloader in process 87559
........

Finished in 0.27244 seconds (files took 0.20924 seconds to load)
8 examples, 0 failures
```

Huzzah! We have our custom filter. All we had to do was define a custom filter and then whittle down the scope as needed. Not to shabby.

## Pagination

Pagination (like filtering) is a somewhat interesting topic in the JSON:API world.  Neither have a spec-enforced implementation. Since we have been using Graphiti though, let's see what that library provides.

```
http://localhost:3000/api/v1/projects?page[size]=1&page[number]=1
```

It really is just that simple. In order to paginate across a resource all you need to do is pass page size and page number. `size` is the number of records per "page", and `number` is the desired page number. There is one small gotcha here. Let's say our initial request to the projects endpoint looked like this:

```bash
http://localhost:3000/api/v1/projects?page[size]=1&page[number]=1
```

This allows us to grab the first page, but we don't see how many pages there are. If we wanted to build client side pagination, we would be scratching our chins right about now. Have no fear though there is a way! We just have to add a little secret sauce to our request.

```bash
http://localhost:3000/api/v1/projects?page[number]=1&page[size]=1&stats[total]=count
```

The trick here is the `stats[total]=count` on the end of the request. This tells Graphiti to return the total number of records. As you might expect it does respect filters. The response looks a little something like below.

```json
"meta": {
    "stats": {
      "total": {
        "count": 3
      }
    }
  }
```

Now we know how to use Graphiti to filter and paginate across data sets! But we need more from our API. We need it to be performant and not require many requests to create or fetch larger data sets. Let's see how JSON:API and Graphiti handle that.

## Nested Writes (side-posting)

If you are familiar with REST, you know one of the biggest downsides is that you can only manipulate one resource at a time. Using an API does not have to be that painful. Imagine we wanted to create a todo list with todo items for a project. But the kicker is we want to do it in just one request.

We would still do a post to the todo lists endpoint.

```
POST http://localhost/api/v1/todo_lists
```

But when we constructed the "body" of the POST request, we need to do a couple of special things. First, under the `relationships` section we need to specify our three todo list items. The 2 unique things here are the attributes "temp-id" and "method". The `temp-id` is just a placeholder that uniquely identifier an object. The `method` parameter is what tells Graphiti what to do to the relationship. The available options are "create", "update", "disassociate", and "destroy".

To pass the details of these records over we need to create an `included` section. This section lives at the same level as `data`. Referencing the JSON below you will see that we reuse the content from the `todo_list_items` section under `relationships`. But now we can use `attributes` to pass over the details of our new todos.

```json
{
  "data": {
    "attributes": {
      "name": "Villians to Smite"
    },
    "relationships": {
      "project": {
        "data": {
          "id": 1,
          "type": "projects"
        }
      },
      "todo_list_items": {
        "data": [
          {
            "temp-id": "111",
            "type": "todo_list_items",
            "method": "create"
          },
          {
            "temp-id": "222",
            "type": "todo_list_items",
            "method": "create"
          },
          {
            "temp-id": "333",
            "type": "todo_list_items",
            "method": "create"
          }
        ]
      }
    },
    "type": "todo_lists"
  },
  "included": [
    {
      "attributes": {
        "content": "write a blog post"
      },
      "temp-id": "111",
      "type": "todo_list_items"
    },
    {
      "attributes": {
        "content": "edit it"
      },
      "temp-id": "222",
      "type": "todo_list_items"
    },
    {
      "attributes": {
        "content": "publish it"
      },
      "temp-id": "333",
      "type": "todo_list_items"
    }
  ]
}
```

This really only scratches the surface of what is possible with JSON:API and Graphiti. You can read more [here](https://www.graphiti.dev/guides/concepts/resources#sideposting) about all the cool things you can do with just one request to your next API.

## Querying Data Graphs

There is just one last thing I would like to talk to you all about today. With GraphQL all the rage and REST looking pretty dusty it is awfully tempting to just go with the trend and use GraphQL. Don't just jump on the bandwagon though! JSON:API and Graphiti query data really expressively and with high levels of complexity. But don't just take my word for it. Let's look at a rather complex example. Imagine that we wanted to get the entire object graph.

```
http://localhost/api/v1/projects?include=todo_lists,todo_lists.todo_list_items
```

The response we get should look like:

```json
{
  "data": [
    {
      "id": "2",
      "type": "projects",
      "attributes": {
        "name": "Project 2",
        "description": "Something cool",
        "purpose": "To organize the project",
        "status": "active"
      },
      "relationships": {
        "todo_lists": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_lists?filter[project_id]=2"
          },
          "data": []
        },
        "todo_list_items": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_list_items?filter[project_id]=2"
          }
        }
      }
    },
    {
      "id": "3",
      "type": "projects",
      "attributes": {
        "name": "Project 3",
        "description": "Something cool",
        "purpose": "To organize the project",
        "status": "active"
      },
      "relationships": {
        "todo_lists": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_lists?filter[project_id]=3"
          },
          "data": [
            {
              "type": "todo_lists",
              "id": "1"
            },
            {
              "type": "todo_lists",
              "id": "2"
            }
          ]
        },
        "todo_list_items": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_list_items?filter[project_id]=3"
          }
        }
      }
    },
    {
      "id": "1",
      "type": "projects",
      "attributes": {
        "name": "updated name",
        "description": "Something cool",
        "purpose": "To organize the project",
        "status": "active"
      },
      "relationships": {
        "todo_lists": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_lists?filter[project_id]=1"
          },
          "data": [
            {
              "type": "todo_lists",
              "id": "4"
            },
            {
              "type": "todo_lists",
              "id": "5"
            }
          ]
        },
        "todo_list_items": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_list_items?filter[project_id]=1"
          }
        }
      }
    }
  ],
  "included": [
    {
      "id": "1",
      "type": "todo_lists",
      "attributes": {
        "name": "new name"
      },
      "relationships": {
        "project": {
          "links": {
            "related": "http://localhost:3000/api/v1/projects/3"
          }
        },
        "todo_list_items": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_list_items?filter[todo_list_id]=1"
          },
          "data": [
            {
              "type": "todo_list_items",
              "id": "6"
            }
          ]
        }
      }
    },
    {
      "id": "2",
      "type": "todo_lists",
      "attributes": {
        "name": "Hop Rod Rye 2"
      },
      "relationships": {
        "project": {
          "links": {
            "related": "http://localhost:3000/api/v1/projects/3"
          }
        },
        "todo_list_items": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_list_items?filter[todo_list_id]=2"
          },
          "data": []
        }
      }
    },
    {
      "id": "4",
      "type": "todo_lists",
      "attributes": {
        "name": "Villians to Smite"
      },
      "relationships": {
        "project": {
          "links": {
            "related": "http://localhost:3000/api/v1/projects/1"
          }
        },
        "todo_list_items": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_list_items?filter[todo_list_id]=4"
          },
          "data": [
            {
              "type": "todo_list_items",
              "id": "1"
            }
          ]
        }
      }
    },
    {
      "id": "5",
      "type": "todo_lists",
      "attributes": {
        "name": "Villians to Smite"
      },
      "relationships": {
        "project": {
          "links": {
            "related": "http://localhost:3000/api/v1/projects/1"
          }
        },
        "todo_list_items": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_list_items?filter[todo_list_id]=5"
          },
          "data": [
            {
              "type": "todo_list_items",
              "id": "2"
            },
            {
              "type": "todo_list_items",
              "id": "4"
            },
            {
              "type": "todo_list_items",
              "id": "5"
            }
          ]
        }
      }
    },
    {
      "id": "6",
      "type": "todo_list_items",
      "attributes": {
        "content": "do the dishes",
        "complete": false
      },
      "relationships": {
        "todo_list": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_lists/1"
          }
        }
      }
    },
    {
      "id": "1",
      "type": "todo_list_items",
      "attributes": {
        "content": "break the dishes!",
        "complete": false
      },
      "relationships": {
        "todo_list": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_lists/4"
          }
        }
      }
    },
    {
      "id": "2",
      "type": "todo_list_items",
      "attributes": {
        "content": "write a blog post",
        "complete": false
      },
      "relationships": {
        "todo_list": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_lists/5"
          }
        }
      }
    },
    {
      "id": "4",
      "type": "todo_list_items",
      "attributes": {
        "content": "edit it",
        "complete": false
      },
      "relationships": {
        "todo_list": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_lists/5"
          }
        }
      }
    },
    {
      "id": "5",
      "type": "todo_list_items",
      "attributes": {
        "content": "publish it",
        "complete": false
      },
      "relationships": {
        "todo_list": {
          "links": {
            "related": "http://localhost:3000/api/v1/todo_lists/5"
          }
        }
      }
    }
  ],
  "meta": {}
}
```

Boom! Just like that we just returned all of our projects, with their todo lists, and todo items. This is called "side-loading" in Graphiti. Essentially it allows a client to request any node(s) in the object graph and dumps them into the `included` section if there are any that exist. There are some gotchas in here around pagination, but if you are interested you can read more [here](https://www.graphiti.dev/guides/concepts/resources#deep-queries).

## Wrap Up

There is so much more power like filtering "side-loaded" relationships, sorting, and more. However, this should be a nice introduction to the flexibility and power that JSON:API and Graphiti provide. Thanks again for sticking with me. If you found this content valuable please do me a kindness by tweeting about it or dropping a reaction below. If you found any grammatical errors or have questions drop a comment and I'll address it. After all this content is for YOU and I want you to get as much value as possible from it.

Cheers!
