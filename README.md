ParseResource 
=============

[![Build Status](https://secure.travis-ci.org/adelevie/parse_resource.png)](http://travis-ci.org/adelevie/parse_resource) [![Code Climate](https://codeclimate.com/github/adelevie/parse_resource.png)](https://codeclimate.com/github/adelevie/parse_resource)


ParseResource makes it easy to interact with Parse.com's REST API. It adheres to the ActiveRecord pattern. ParseResource is fully ActiveModel compliant, meaning you can use validations and Rails forms.

Ruby/Rails developers should feel right at home.

If you're used to `Post.create(:title => "Hello, world", :author => "Octocat")`, then this is for you.

Features
---------------
*   ActiveRecord-like API, almost no learning curve
*   Validations
*   Rails forms and scaffolds **just work**

Use cases
-------------
*   Build a custom admin dashboard for your Parse.com data
*   Use the same database for your web and native apps
*   Pre-collect data for use in iOS and Android apps


Installation
------------

Include in your `Gemfile`:

```ruby
gem "kaminari" # optional for pagination support
gem "parse_resource", "~> 1.8.0"
```

Or just gem install:

```ruby
gem install kaminari # optional for pagination support
gem install parse_resource
```

Create an account at Parse.com. Then create an application and copy the `app_id` and `master_key` into a file called `parse_resource.yml`. If you're using a Rails app, place this file in the `config` folder.

```yml
development:
  app_id: 1234567890
  master_key: abcdefgh

test:
  app_id: 1234567890
  master_key: abcdefgh

production:
  app_id: 1234567890
  master_key: abcdefgh
```

If you keep `parse_resource.yml` in `.gitignore`, ParseResource will alternatively look for the api keys in environment variables. If using Heroku you can easily set your api keys in the Heroku environment using:

```
heroku config:set app_id=1234567890
heroku config:set master_key=abcdefgh
```

You can create separate Parse databases if you want. If not, include the same info for each environment.

In a non-Rails app, include this somewhere (preferable in an initializer):


```ruby
ParseResource::Base.load!("your_app_id", "your_master_key")
```


Usage
-----

Create a model:

```ruby
class Post < ParseResource::Base
  fields :title, :author, :body

  validates_presence_of :title
end
```

If you are using version `1.5.11` or earlier, subclass to just `ParseResource`--or just update to the most recent version.

Creating, updating, and deleting:

```ruby
p = Post.new

# validations
p.valid? #=> false 
p.errors #=> #<ActiveModel::Errors:0xab71998 ... @messages={:title=>["can't be blank"]}> 
p.title = "Introducing ParseResource" #=> "Introducing ParseResource" 
p.valid? #=> true 

# setting more attributes, then saving
p.author = "Alan deLevie" 
p.body = "Ipso Lorem"
p.date = Time.now
p.save #=> true

# checking the id generated by Parse's servers
p.id #=> "QARfXUILgY" 
p.updated_at #=> nil 
p.created_at #=> "2011-09-19T01:32:04.973Z" # does anybody want this to be a DateTime object? Let me know.

# updating
p.title = "[Update] Introducing ParseResource"
p.save #=> true
p.updated_at #=> "2011-09-19T01:32:37.930Z" # more magic from Parse's servers

# destroying an object
p.destroy #=> true 
p.title #=> nil
```

Finding:

```ruby
posts = Post.where(:author => "Arrington")
# the query is lazy loaded
# nothing gets sent to the Parse server until you run #all, #count, or any Array method on the query 
# (e.g. #first, #each, or #map)

posts.each do |post|
  "#{post.title}, by #{post.author}"
end

posts.map {|p| p.title} #=> ["Unpaid blogger", "Uncrunched"]

id = "DjiH4Qffke"
p = Post.find(id) #simple find by id

# ActiveRecord style find commands
Post.find_by_title("Uncrunched") #=> A Post object
Post.find_all_by_author("Arrington") #=> An Array of Posts

# batch save an array of objects
Post.save_all(array_of_objects)

# destroy all objects, updated to use Parse batch destroy
Post.destroy_all(array_of_objects)

# you can chain method calls, just like in ActiveRecord
Post.where(:param1 => "foo").where(:param2 => "bar").all


# limit the query
posts = Post.limit(5).where(:foo => "bar")
posts.length #=> 5

# get a count
Post.where(:bar => "foo").count #=> 1337

```

Pagination with [kaminari](https://github.com/amatsuda/kaminari):

```ruby
# get second page of results (default is 25 per page)
Post.page(2).where(:foo => "bar")

# get second page with 100 results per page
Post.page(2).per(100).where(:foo => "bar")

```

Users

Note: Because users are special in the Parse API, you must name your class User if you want to subclass ParseUser.

```ruby
# app/models/user.rb
class User < ParseUser
  # no validations included, but feel free to add your own
  validates_presence_of :username
  
  # you can add fields, like any other kind of Object...
  fields :name, :bio
  
  # but note that email is a special field in the Parse API.
  fields :email
end

# create a user
user = User.new(:username => "adelevie")
user.password = "asecretpassword"
user.save #=> true
# after saving, the password is automatically hashed by Parse's server
# user.password will return the unhashed password when the original object is in memory
# from a new session, User.where(:username => "adelevie").first.password will return nil

# check if a user is logged in
User.authenticate("adelevie", "foooo") #=> false
User.authenticate("adelevie", "asecretpassword") #=> #<User...>


# A simple controller to authenticate users
class SessionsController < ApplicationController
  def new
  end
  
  def create
    user = User.authenticate(params[:username], params[:password])
    if user
      session[:user_id] = user.id
      redirect_to root_url, :notice => "logged in !"
    else
      flash.now.alert = "Invalid username or password"
      render "new"
    end
  end
  
  def destroy
    session[:user_id] = nil
    redirect_to root_url, :notice => "Logged out!"
  end

end

```

If you want to use parse_resource to back a simple authentication system for a Rails app, follow this [tutorial](http://asciicasts.com/episodes/250-authentication-from-scratch), and make some simple modifications.

Installations

Note: Because [Installations](https://parse.com/docs/rest#installations), are special in the Parse API, you must name your class Installation if you want to manipulate installation objects.

```ruby
class Installation < ParseResource::Base
  fields :appName, :appVersion, :badge, :channels, :deviceToken, :deviceType,
         :installationId, :parseVersion, :timeZone
end
```

GeoPoints

```ruby
class Place < ParseResource::Base
  fields :location
end

place = Place.new
place.location = ParseGeoPoint.new :latitude => 34.09300844216167, :longitude => -118.3780094460731
place.save
place.location.inspect #=> #<ParseGeoPoint:0x007fb4f39c7de0 @latitude=34.09300844216167, @longitude=-118.3780094460731>


place = Place.new
place.location = ParseGeoPoint.new
place.location.latitude = 34.09300844216167
place.location.longitude = -118.3780094460731
place.save
place.location.inspect #=> #<ParseGeoPoint:0x007fb4f39c7de0 @latitude=34.09300844216167, @longitude=-118.3780094460731>

server_place = Place.find(place.objectId)
server_place.location.inspect #=> #<ParseGeoPoint:0x007fb4f39c7de0 @latitude=34.09300844216167, @longitude=-118.3780094460731>
server_place.location.latitude #=> 34.09300844216167
server_place.location.longitude #=> -118.3780094460731
```

Querying by GeoPoints

```ruby
Place.near(:location, [34.09300844216167, -118.3780094460731], :maxDistanceInMiles => 10).all
Place.near(:location, [34.09300844216167, -118.3780094460731], :maxDistanceInKilometers => 10).all
Place.near(:location, [34.09300844216167, -118.3780094460731], :maxDistanceInRadians => 10/3959).all
Place.within_box(:location, [33.81637559726026, -118.3783150233789], [34.09300844216167, -118.3780094460731]).all
```

DEPRECATED
Associations

```ruby
class Post < ParseResource::Base
  # As with ActiveRecord, associations names can differ from class names...
  belongs_to :author, :class_name => 'User'
  fields :title, :body
end

class User < ParseUser
  # ... but on the other end, use :inverse_of to complete the link.
  has_many :posts, :inverse_of => :author
  field :name
end

author = Author.create(:name => "RL Stine")
post1 = Post.create(:title => "Goosebumps 1")
post2 = Post.create(:title => "Goosebumps 2")

# assign from parent class
author.posts << post1
author.posts << post2 

# or assign from child class
post3 = Post.create(:title => "Goosebumps 3")
post3.author = author
post3.save #=> true

# relational queries
posts = Post.include_object(:author).all
posts.each do |post|
	puts post.author.name
	# because you used Post#include_object, calling post.title won't execute a new query
	# this is similar to ActiveRecord's eager loading
end

# fetch users through a relation on posts named commenters
post = Post.first
users = User.related_to(post, :commenters)
```

File Upload

```ruby
  @post = Post.first()
  result = Post.upload(uploaded_file.tempfile, uploaded_file.original_filename, content_type: uploaded_file.content_type)
  @post.thumbnail = {"name" => result["name"], "__type" => "File"}
```

Custom Getters and Setters

```ruby
  def name
    val = get_attribute("name")
    # custom getter actions here
    val
  end

  def name=(val)
    # custom setter actions to val here
    set_attribute("name", val)
  end
```

Documentation
-------------
[Here](http://rubydoc.info/gems/parse_resource/)

To-do
--------------
*   User authentication
*   Better documentation
*   ~~Associations~~
*   Callbacks
*   Push notifications
*   Better type-casting
*   HTTP request error handling

User authentication is my top priority feature. Several people have specifically requested it, and Parse just began exposing [User objects in the REST API](https://www.parse.com/docs/rest#users).

Let me know of any other features you want.


Contributing to ParseResource
-----------------------------

*   Check out the latest master to make sure the feature hasn't been implemented or the bug hasn't been fixed yet
*   Check out the issue tracker to make sure someone already hasn't requested it and/or contributed it
*   Fork the project
*   Start a feature/bugfix branch
*   Commit and push until you are happy with your contribution
*   Make sure to add tests for it. This is important so I don't break it in a future version unintentionally.
*   Create `parse_resource.yml` in the root of the gem folder. Using the same format as `parse_resource.yml` in the instructions (except only creating a `test` environment, add your own API keys.
*   Please try not to mess with the Rakefile, version, or history. If you want to have your own version, or is otherwise necessary, that is fine, but please isolate to its own commit so I can cherry-pick around it.


Copyright
---------

Copyright (c) 2013 Alan deLevie. See LICENSE.txt for
further details.

