---
layout: post
title: The friendly union of rails-base-api and active-admin
date: 2015-09-25 00:00:00
---

What do you do when a simple API is needed as soon as possible? Maybe your new awesome iOS-application is ready for alpha-test, but due to the lack of API it has only harcoded data? Perhaps you have just written an amazing front-end framework and need the endpoints to investigate it properly? How do we make the back-end strong and stable in such cases?

For API we usually use [rails-base-api](https://github.com/fs/rails-base-api). Without preparation it includes some of useful gems such as `rails-api`, `active_model_serializers` and many convinient tools for testing. Also it truncates unused functionality for rails since you don't need the entire middleware stack. For instance, the template generation. Thus, the whole application is a bit more lightweight and faster. So you don't have to figure out which configuration you should use to create your API. Everything you need to do is type `bin/setup` and start bringing to bear your api-endpoints. Sounds fantastic, doesn't it?

However, often API endpoints are not enough for live projects.

Obviously, a live application has to provide an interface to work with information from the database. To save time I’ll be looking at one of the popular options available - [ActiveAdmin](https://github.com/activeadmin/activeadmin). ActiveAdmin is a gem that gives you a very good-looking backend panel for your application.

Unfortunately, we are not gonna be able to simply add the `gem "activeadmin"` to our Gemfile, because rails-base-api doesn't have some crucial parts for serve HTML resources. The set of certain actions needs to be accomplished to connect gem `activeadmin`. Let me share with you some of them that work. The method described below suits for versions of `ruby "2.2.2"` and `rails "4.2.3"`. I hope you find it useful.

Firstly, update your Gemfile by adding two gems:

```ruby
gem "activeadmin", github: "activeadmin"
gem "sass-rails"
```

Then install gems and initialize active_admin:

```ruby
rails g active_admin:install
```

Now let's rename `application_controller.rb` to `base_api_controller.rb`. Now it's basis for api-controllers. `ApplicationController` shoud be used for ActiveAdmin main controller and, consequently, must be inherited from `ActionController::Base` instead of `ActionController::API`:

```ruby
class ApplicationController < ActionController::Base
end
```

To create first admin_user:

```ruby
rake db:migrate
rake db:seed
```

We need to use a number of other middlewares that are not built in `rails-api` by default. For that purposes add these lines to `application.rb`:

```ruby
# to support the flash mechanism in ActionController.
config.middleware.use ActionDispatch::Flash
# add encrypted cookies. It requires for the authentication with devise.
config.middleware.use ActionDispatch::Cookies
config.middleware.use ActionDispatch::Session::CookieStore
```

Voila! It's ready to roll. From this point be free to use activeadmin [documentation](https://github.com/activeadmin/activeadmin/tree/master/docs#activeadmin-documentation) for the interface customization.

Now we are happy owners of a a solid administrative interface, ready to begin entering information. No need to hook up with full rails stack. As a bonus we get a few hours saved by not pouring additional effort into creating a similar UI. Enjoy!
