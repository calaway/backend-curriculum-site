---
layout: page
title: Building an Internal API
subheading:
length: 90
tags: apis, testing, requests, rails
---

## Install Rails

Rails can take a while to install, so if you haven't already let's get this going first.

```sh
$ gem install rails
```

You can confirm it installed correctly by running

```sh
$ rails --version
```

If the output is something like `Rails 5.x.x` then you're good to go.

## Learning Goals

* Understand how an internal API works at a conceptual level
  * Q: What is an API in the context of web development?
  * A: At the most basic level an API wraps a database and provides it's users to interface with the database via HTTP requests
  * Q: Why might we decide to expose information in a database we control through an API?
  * A: To save the user the headache of making database calls directly
* Understand what it means to build a CRUD app and why that is significant
* Understand the 7 restful routes and the 5 we will use to build an API
  * Write out the [7 restful routes](http://guides.rubyonrails.org/routing.html#crud-verbs-and-actions) and discuss the relationship with CRUD
* Understand how to create a Postgres database, add items to it, retrieve items from it, update items, and destroy items. This corresponds to the Create, Read, Update, & Destroy CRUD acronym.
* Understand the MVC Architecture and why we don't need the View component in an API
* Build our app via test driven development (TDD), by creating request specs to cover an internal API
* Feel comfortable writing request specs that deal with different HTTP verbs (GET, POST, PUT, DELETE)
  * Q: What do we need to test in an API?
  * A: We need one test for each API endpoint we expose to the users

## Warmup with a 10 minute blog

An API is a specific type of web app. So before we jump in let's witness the magic of Rails by building a standard web app in the form of a blog. We'll loosely follow the official Rails documentation on creating a new rails project [here](http://guides.rubyonrails.org/v3.2.8/getting_started.html#creating-a-new-rails-project).

We start by creating a new Rails app, changing directories into the project, and initializing the database for the app.

```sh
$ rails new blog -T -d postgresql
$ cd blog
$ rails db:create
```

Now we can really lean into the magic of rails by using its scaffold generator to build a CRUD app. Our 'resource' will be a blog post. Following that we will run the migrate command to add our blog posts table to our database.

```sh
$ rails generate scaffold Post name:string title:string content:text
$ rails db:migrate
```

Rails scaffolding generator is the cheater's way to have Rails build out all 7 restful routes in a CRUD app with that one command. Let's run `$ rails routes` to see what pages it generated for us. To see them in a browser you'll want to spin up the Rails server with the command `$ rails server`. This will start our app on `http://localhost:3000`. To see our blog navigate to the url [http://localhost:3000/posts](http://localhost:3000/posts). Go ahead and play around with it by creating a new post or two, viewing them, editing one, and deleting one.

That's it! You have a new blog!

Now change directories out of this project, delete it if you like and let's get started on our API.

## Procedure

### Overview

By the time we finish we will have written a test for each of the 5 restul routes that apply to an API and will have built out the 5 endpoints to make each test pass. Our endpoints will be:

* Api::V1::ItemsController#index
* Api::V1::ItemsController#show
* Api::V1::ItemsController#create
* Api::V1::ItemsController#update
* Api::V1::ItemsController#destroy

### Testing Tools

* `get 'api/v1/items'`: submits a get request to your application
* `response`: captures the response to a given request
* `JSON.parse(response)`: parses a JSON response

### Controller Tools

* `render`: tells your controller what to render as a response
* `json: Item.all`: hash argument for render - converts Item.all to valid JSON

## 0. RSpec Setup

Let's start by creating a new Rails project. If you are creating an api only Rails project, you can append `--api` to your rails new line in the command line.
Read [section 3 of the docs](http://edgeguides.rubyonrails.org/api_app.html) if you want to see how an api-only rails project is configured.

```sh
$ rails new my_api -T -d postgresql --api
$ cd my_api
$ rails db:create
```

Add `gem 'rspec-rails'` & `gem 'pry-rails'` to your Gemfile within `group :development, :test`, then bundle and generate the basic RSpec setup.

```sh
$ bundle
$ rails g rspec:install
```

## 1. Creating Our First Test

Now that our configuration is set up, we can start test driving our code. First, let's set up the test file. In true TDD form, we need to create the structure of the test folders ourselves. Even though we are going to be creating controller files for our api, users are going to be sending HTTP requests to our app. For this reason, we are going to call these specs `requests` instead of `controller specs`. Let's create our folder structure.

```sh
$ mkdir -p spec/requests/api/v1
$ touch spec/requests/api/v1/items_request_spec.rb
```

Note that we are namespacing under `/api/v1`, which is a best practice when creating an API because it leave us room to add later versions on our API while keeping the initial version intact. This is how we are going to namespace our controllers, so we want to do the same in our tests.

Note also that we are deciding to use 'items' as the resource that our API is serving up. You can really use any noun that would make sense to store in a database, e.g. users, photos, blog posts, restaurant reviews, et cetera.

At the beginning of our test, we want to set up our data. We'll create a few items in our database that we can then request back via our API. We then want to make the request that a user would be making. We want a `get` request to `api/v1/items` and we would like to get json back. At the end of the test we want to assert that the response was a success.

```rb
# spec/requests/api/v1/items_request_specs_spec.rb
require 'rails_helper'

RSpec.describe "Items API", type: :request do
  it "index: returns a list of all items" do
    3.times do |n|
      Item.create(name: "Item_#{n}", description: "Description of item #{n}")
    end

    get '/api/v1/items'

    expect(response).to have_http_status(200)
  end
end
```

This is a very basic test, that only tests that we get something/anything back from the API call. Well fill in the test more when this part is passing.

## 2. Creating Our First Model & Migration

Let's make the test pass!

When we run the command `$ rspec` the first error that we should receive is

```sh
Failure/Error: Item.create(name: "Item_#{n}", description: "Description of item #{n}")
NameError: uninitialized constant Item
```

This is because we have not created a our Item model yet. Rails gives us a generator to create the model and the migration that we'll use to add the model table to the database.

Let's generate a model.

```sh
$ rails g model Item name:string description:text
```

Take a look at the output of this command to see what the generator created for us. We should have a migration, a model, and a spec for the model. In our case we're testing it via our items_request_spec, so you can delete the model spec with `$ rm spec/models/item_spec.rb`.

Now let's migrate!

```sh
$ rails db:migrate
== 20160229180616 CreateItems: migrating ======================================
-- create_table(:items)
   -> 0.0412s
== 20160229180616 CreateItems: migrated (0.0413s) =============================
```

## 3. Controller#Action - Api::V1::ItemsController#index

The rule with TDD is to make the smallest change that will fix the current error and then run the tests again ... rince and repeat. So let's run `$ rspec` again.

We should get the error `ActionController::RoutingError: No route matches [GET] "/api/v1/items"`
This is because we haven't created a route yet. Without a route, when a user sends an HTTP request to `/api/v1/items` we haven't told our app how to handle that request. So let's create the route! Keep in mind that we namespaced our API routes in the test directory under `api/v1`.

```rb
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      get '/items', to: 'items#index'
    end
  end
end
```

This should fix our previous error, so it's time to run rspec agian to chase down the next one.

```sh
ActionController::RoutingError: uninitialized constant Api
```

This tells us that our API request is looking for a controller called Api. A controller is just a special type of class located under `/app/controllers`. In our case it will be under `/app/controllers/api/v1`. So let's create the controller!

```sh
$ mkdir -p app/controllers/api/v1
$ touch app/controllers/api/v1/items_controller.rb
```

We can add the following to the controller we just made:

```rb
# app/controllers/api/v1/items_controller.rb
class Api::V1::ItemsController < ApplicationController
end
```

Now we run our test again and get `The action 'index' could not be found for Api::V1::ItemsController`. An 'action' simply means a method within the controller. To that effect, when we directed our route in the routes file to `items#index` we call `items#index` a controller-action.

So let's add the index action to our controller and then run our test again.

```rb
# app/controllers/api/v1/items_controller.rb
class Api::V1::ItemsController < ApplicationController
  def index
  end
end
```

Great! We are successfully getting a response! But we aren't actually getting any data. Without any data or templates, Rails 5 API will respond with `Status 204 No Content`. Our test is requiring a 200 status code.

Now lets see if we can actually get some data.

```rb
# class Api::V1::ItemsController < ApplicationController
class Api::V1::ItemsController < ApplicationController
  def index
    render plain: 'Hello World!'
  end
end
```

Run rspec again and ... we have a passing test!

Now let's fill in our test to make sure we're retuning what we actually want from our API.

```rb
# spec/requests/api/v1/items_request_spec.rb
require 'rails_helper'

RSpec.describe "Items API", type: :request do
  it "index: returns a list of all items" do
    3.times do |n|
      Item.create(name: "Item_#{n}", description: "Description of item #{n}")
    end

    get '/api/v1/items'
    items_response = JSON.parse(response.body)

    expect(response).to have_http_status(200)
    expect(items_response.count).to eq(3)
    expect(items_response.first['name']).to eq('Item_0')
    expect(items_response.last['description']).to eq('Description of item 2')
  end
end
```

When we run our tests again, we get a `JSON::ParserError`. This just means that the data we returned is not formatted as JSON.

Let's render some JSON from our controller.

```rb
# app/controllers/api/v1/items_controller.rb
class Api::V1::ItemsController < ApplicationController
  def index
    render json: Item.all
  end
end
```

And ... our test is passing again!

Let's take a closer look at the response. Put a pry on line ten in the test, right below where we make the request.

If you just type `response` you can take a look at the entire response object. We care about the response body. If you enter `response.body` you can see the data that is returned from the endpoint.

The data we got back is json, and we need to parse it to get a Ruby object. Try entering `JSON.parse(response.body)`. As you see, the data looks a lot more like Ruby after we parse it. Now that we have a Ruby object, the assertions we made about it are passing.

### Hitting our API endpoint in development

Okay, so we've built an API endpoint and we know it's working in our test environment. Let's make sure it works when the rubber meets the road by trying to hit it on the development environment. First, jump into the Rails console and add a few items to the database. Then spin up your Rails server to expose the endpoint we created.

```sh
$ rails console
```

```rb
[1] pry(main)> 3.times do |n|
[1] pry(main)*   Item.create(name: "Item_#{n}", description: "Description of item #{n}")
[1] pry(main)* end
[2] pry(main)> Item.count # => 3
[3] pry(main)> exit
```

```sh
$ rails server
```

Now point your browser to [http://localhost:3000/api/v1/items](http://localhost:3000/api/v1/items) and you should see your three items with their descriptions. If you want to see the output in a more organized format, I recommend installing a browser extension to take care of that. I use [JSONView](https://chrome.google.com/webstore/detail/jsonview/chklaanhfefbnpoihckbnefhakgolnmc) for Chrome and am pretty happy with it.

Make note that the nature of this transaction requires both a client and a server. In this case your computer is playing both roles. Your terminal is providing the server that anyone (who can connect to it) can make a request from. Then your browser is sending a request to the server and the server returns the information requested by the browser.

Now we're ready to move on to the next route: show.

## 4. Controller#Action - Api::V1::ItemsController#show

Now we are going to test drive the `/api/v1/items/:id` endpoint. From the `show` action, we want to return a single item.

First, let's write a second test within the same feature. As you can see, we have added a key `id` in the request:

```rb
# spec/requests/api/v1/items_request_spec.rb
it 'show: returns a single item by id' do
  item = Item.create(name: 'Flux Capacitor',
                     description: 'Time travel device running on plutonium')
  id = item.id

  get "/api/v1/items/#{id}"
  item_response = JSON.parse(response.body)

  expect(response).to have_http_status(200)
  expect(item_response['id']).to eq(id)
  expect(item_response['name']).to eq('Flux Capacitor')
end
```

Try to test drive the implementation before looking at the code below. Remember the pattern:
1. Run the test & get a failure message.
2. Fix the smallest thing in the code to fix the immediate error.
3. Repeat steps 1 & 2 until the test is passing.

Run the tests and the first error we get is: `ActionController::RoutingError: No route matches [GET] "/api/v1/items/[some_number]"`.

Let's update our routes.

```rb
# config/routes.rb
namespace :api do
  namespace :v1 do
    get '/items',     to: 'items#index'
    get '/items/:id', to: 'items#show'
  end
end
```

Run the tests and... `The action 'show' could not be found for Api::V1::ItemsController`.

Add the action and declare what data should be returned from the endpoint:

```rb
# app/controllers/api/v1/items_controller.rb
class Api::V1::ItemsController < ApplicationController
  def index
    render json: Item.all
  end

  def show
    render json: Item.find(params[:id])
  end
end
```

Run the tests and... we should have two passing tests. If you want a better understanding of the `params` variable, throw a pry in on the line after `def show` and play around with it.

### Hitting our API endpoint in development

To hit our new endpoint in the development environment the only difference is to add a number to the end of the url. First, to find out what index numbers were assigned in your database, jump into `rails console` and run `Item.all` to view the ids of the items you created before (this will likely just be 1, 2, & 3). Then point your browser at [http://localhost:3000/api/v1/items/1](http://localhost:3000/api/v1/items/1). Your API should return just the one item this time.

Now let's step it up a notch and move from GET requests to a POST request.

## 5. Controller#Action - Api::V1::ItemsController#create

At this point we've created 2 of the 5 restful routes (index & show are done, create, update, & destroy are left). Since the setup took a while I'd say we're at least halfway done.

For 'create', let's start with the test. Since we are creating a new item, we need to pass data for the new item via the HTTP request. We can do this easily by adding the params as a key-value pair. Also note that we swapped out the `get` in the request for a `post` since we are creating data.

Also note that we aren't parsing the response to access the last item we created, we can simply query for the last Item record created.

```rb
# spec/requests/api/v1/items_request_spec.rb
it 'create: creates a new item' do
  item_params = { name: 'Post Holer', description: 'It\'s for digging holes ... for posts' }

  post '/api/v1/items', params: item_params
  item = Item.last

  assert_response :success
  expect(response).to be_success
  expect(item.name).to eq(item_params[:name])
end
```

Run the test and you should get `ActionController::RoutingError:No route matches [POST] "/api/v1/items"`

First, we need to add the route.

```rb
# config/routes.rb
namespace :api do
  namespace :v1 do
    get  '/items',     to: 'items#index'
    get  '/items/:id', to: 'items#show'
    post '/items',     to: 'items#create'
  end
end
```

The next error should look familiar by now and should indicate that `The action 'create' could not be found for Api::V1::ItemsController`. So we need to add the create action (method) to the ItemsController.

```rb
# app/controllers/api/v1/items_controller.rb
def create
end
```

Run the tests... and the test fails. You should get `NoMethodError: undefined method 'name' for nil:NilClass`. That's because we aren't actually creating anything yet.

We are going to create an item with the incoming params. Again, I find it helpful to throw a debugger (read: pry) in after `def create` and look closer at the `params` variable to get a better understanding.

```rb
# app/controllers/api/v1/items_controller.rb
def create
  item = Item.create(name: params[:name], description: params[:description])
  render json: item
end
```

Run the tests and we should have 3 passing tests.

## 6. Controller#Action - Api::V1::ItemsController#update

Like before, let's add a test.

This test looks very similar to the previous one we wrote. Note that we aren't making assertions about the response, instead we are accessing the item we updated from the database to make sure it actually updated the record.

```rb
# spec/requests/api/v1/items_request_spec.rb
it "update: updates an existing item" do
  id = Item.create(name: 'Caterpillar', description: 'Turns into a butterfly').id
  previous_name = Item.last.name
  item_params = { name: 'Butterfly' }

  put "/api/v1/items/#{id}", params: item_params
  item = Item.find(id)

  expect(response).to be_success
  expect(item.name).to_not eq(previous_name)
  expect(item.name).to eq('Butterfly')
end
```

Try to test drive the implementation before looking at the code below.
---

```rb
# config/routes.rb
namespace :api do
  namespace :v1 do
    get  '/items',     to: 'items#index'
    get  '/items/:id', to: 'items#show'
    post '/items',     to: 'items#create'
    put  '/items/:id', to: 'items#update'
  end
end
```

```rb
# app/controllers/api/v1/items_controller.rb
def update
  item = Item.update(params[:id], name: params[:name])
  render json: item
end
```

## 7. Controller#Action - Api::V1::ItemsController#destroy

Ok, last endpoint to test and implement: destroy!

In this test, the last line in this test is refuting the existence of the item we created at the top of this test.

```rb
# spec/requests/api/v1/items_request_spec.rb
it "destroy: can destroy an item" do
  item = Item.create(name: 'Death Star', description: "That's no moon ... It's a space station")

  expect{ delete "/api/v1/items/#{item.id}" }.to change{ Item.count }.from(1).to(0)

  expect(response).to be_success
  expect{Item.find(item.id)}.to raise_error(ActiveRecord::RecordNotFound)
end
```

You can read about RSpec's [expect change matcher here](https://www.relishapp.com/rspec/rspec-expectations/v/2-0/docs/matchers/expect-change). In our case, `change` will verify the numeric value of `Item.count` before and after the block is run.

Make the test pass.
---

```rb
# config/routes.rb
namespace :api do
  namespace :v1 do
    get    '/items',     to: 'items#index'
    get    '/items/:id', to: 'items#show'
    post   '/items',     to: 'items#create'
    put    '/items/:id', to: 'items#update'
    delete '/items/:id', to: 'items#destroy'
  end
end
```

```rb
# app/controllers/api/v1/items_controller.rb
def destroy
  Item.destroy(params[:id])
end
```

Pat yourself on the back. You just built an API. And with TDD. Huzzah! Now go call a friend and tell them how cool you are.

## Supporting Materials

* [Notes](https://www.dropbox.com/s/zxftnls0at2eqtc/Turing%20-%20Testing%20an%20Internal%20API%20%28Notes%29.pages?dl=0)
* [Getting started with Factory Girl](https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md)
* [Use Factory Girl's Build Stubbed for a Faster Test](https://robots.thoughtbot.com/use-factory-girls-build-stubbed-for-a-faster-test)
* [Building an Internal API Short Tutorial](https://vimeo.com/185342639)
