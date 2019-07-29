---
layout: post
title:  "GraphQL Is Dead, Long Live JSON:API (In Rails)"
date:   2019-07-30
categories: software,json,api,graphql,rails,ruby
---

Before I get started, I'd like to start with a simple disclaimer. GraphQL is not a terrible solution or technology. In my opion however, it sacrifices quite a lot of features and functionality that a RESTful API gives you. It does this in the name of flexibility, but here I'll demonstrate to you that flexibility is not something sacrificed by using a JSON:API approach. Without further ado, let's get into it.

**Note** This article does assume at least some base knowledge of GraphQL and API Design.

## The Setup

Imagine you were tasked with building an API for Basecamp. They wanted you to expose an interface for projects. Users need to authenticate to get a list of projects and projects have members, tasks, documents, and posts. The first feature requested is to allow users to fetch all of their projects and associated data.

## The Prototype

Assuming the schema is already in place with all proper types defined, the query might look something like we have below.

{% highlight ruby %}
query {
  projects($accountId: 1) {
    id,
    name

    projectMembers {
      user {
        id
        name
      }
    }

    todos {
      content
      complete

      assignee {
        name
      }
    }

    documents {
      name
      versionNumber
      url
    }
  }
}
{% endhighlight ruby %}

Using JSON:API with something like [Graphiti](https://www.graphiti.dev) the request would look like this:

{% highlight bash %}
curl https://localhost:3000/api/v1/projects?include=project_members,project_members.user,todos,todos.assignee,documents&filter[account_id][eq]=1&fields[container]=id,name&fields[user]=id,name&fields[todos]=content,complete&fields[documents]=name,version_number,url \
	-H "Content-Type: application/vnd.api+json" \
	-H "Accept: application/vnd.api+json"
{% endhighlight bash %}

The results returned by both APIs are equivalent. Interested in hearing more? Read on. Fetching data is probably where GraphQL shines the most because it allows a client to specify objects, relationships, and "sparse fieldsets". However we can see above that JSON:API can do that too, albeit with slightly less readability. Of course, if we executed the GraphQL request with curl we would see something pretty similar to our JSON:API situation.

Let's move on to where writes, as this is probably the most interesting comparison. We start again by looking at how to do this with GraphQL.

We start by defining the allowed "mutation"

{% highlight ruby %}
mutation CreateTodoForProject($projectId: Integer!, $content: String!, $assigneeId: Integer, $completed: Boolean) {
  createTodo(projectId: $productId, content: $content, assigneeId: $assigneeId, completed: $completed) {
    projectId
    content
    completed
    assigneeId
  }
}
{% endhighlight ruby %}

And then we make our request:

{% highlight json %}
{
  "data": {
    "createTodo": {
      "projectId": 1,
      "content": "This is great content",
      "completed": false,
      "assigneeId": 2
    }
  }
}
{% endhighlight json %}

Well isn't this interesting..? In order to mutate our data we have to tell GraphQL how to allow it. That seems cumbersome, I wonder how JSON:API handles it.

{% highlight json %}
{
  "data": {
    "type":
    "projects", "id": 1,
    "relationships": {
      "todos": [{ "type": "todos", "temp-id": "1233345", "method": "create" }]
    }
  },
  "included": {
    "temp-id": "1233345",
    "type": "todos",
    "attributes": {
      "assignee_id": 2,
      "content": "This is great content",
      "completed": false
    }
  }
}
{% endhighlight json %}

What I find really powerful about this example is the lack of work needed to support it. Frameworks like Graphiti or JSONAPI:Resources understands your data hierachy based on the relationships you define. Once you have that done you can write simple POSTs or complex nested changes. There is no need to maintain mutations and error messages are returned where they are relevant. If the request above was invalid, the response might look something like:

{% highlight json %}
{
  "errors": [{
    "code": "unprocessable_entity",
    "status": "422",
    "title": "Validation Error",
    "detail": "content can't be blank",
    "source": { "pointer": "/data/attributes/content" },
    "meta": {
      "relationship": {
        "attribute": "content",
        "message": "can't be blank",
        "code": "blank",
        "content": "",
        "id": "1233345",
        "type": "todos"
      }
    }
  }]
{% endhighlight json %}

This is pretty nice! We can perform our nested writes and if anything fails you know right where it did. Still not convinced? Ok, let's take a look at how GraphQL might handle this same situation.

{% highlight json %}
{
  "errors": [{
    "message": "Request is invalid.",
    "path": ["createTodo"],
    "state": {
      "content": ["can't be blank"]
    }
  }]
}
{% endhighlight json %}

Hmmm, this is a bit less awesome. We can see that our operation failed, but we don't have a good correlation between our proposed todo object and these errors. This means that on a fundamental level your errors are tied to mutations and not necessarily tied to the objects you are trying to mutate. It makes things like rendering complex nested forms pretty difficult. If you want to read a bit more about mutations and validation check out [this](https://medium.com/@tarkus/validation-and-user-errors-in-graphql-mutations-39ca79cd00bf) excellent article on medium.

Especially in Ruby land, things are not quite where you want them to be. If you want your API to scale well, handle complex authorization schemes, or cache your responses if have to roll your own solution or pay for them. Right now [GraphQL Pro](https://graphql.pro) seems to be the most popular solution to these concerns. If you don't want to pay you can find an [open source caching gem](https://github.com/stackshareio/graphql-cache) and [this](https://github.com/Shopify/graphql-batch) gem from Shopify that aims to make batch operations less painful.

It's really easy to latch onto new and exciting and even popular tech. The great folks at Facebook have done an amazing amount of work to create GraphQL and maintain pretty excellent docs to boot. However, GraphQL isn't the only way to build APIs right now, especially not in Rails. It may not even be the best way. For you more fullstack folks I know what you are thinking...there are lots of tutorials out there on Apollo with React and lots of folks are building things this way. But I encourage you to push past the popularity. Some folks are already hard at work publishing good client libraries to consume JSON:API services. Check these out:

* [Redux](https://github.com/cantierecreativo/redux-bees)
* [JSON:API Client](https://github.com/itsfadnis/jsonapi-client)
* [MobX](https://mobx.reststate.org/)
* [Vuex](https://vuex.reststate.org/)

Hopefully I haven't upset any of you folks who have stuck around to read this engineer's opinion. I'm currently building a JSON:API on Rails with Graphiti that I plan to share with you soon. We will go more in depth with some of the topics we brushed through on here and after that we will dive into the client side and build a modern React application that consumes our API. I'm really excited to show how great building APIs on this tech stack can be. Cheers!
