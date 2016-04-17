---
layout: post
title: How many types of objects your Rails app uses?
description: Use extra layers in your Rails app to keep your code lean and mean.
date: 2016-04-06 00:00:00
---
Ok guys, I know that you've worked/currently working with legacy code. And I pretty sure that you've also faced with dread and horror.

Let me guess what you see when you open a legacy project.

- Huge controllers with enormous methods
- Dozen of conditionals inside views
- Logic that extracted to helpers in order not to polute views. It causes having complex organized methods inside the helpers directory.

The one of achilles heels of Rails is that standard MVC stack is not enough in many cases. Models and controllers start to grew up and violate Single Responsibility Principle.

As a consequence, couple of layers apeared to keep your code lean and mean. Usually they exist in separate directories and provide a wonderful degree of encapsulation. In this artcile I want to describe the most popular of them.

## 1. Parameter Object

It's generally known that developers don't like magic numbers. To avoid them we create constants with the explaining meaning names. The top of class usually contains several of them:

~~~ruby
class Mail < ActiveRecord::Base
  MAX_LETTERS_NUMBER = 10_000
  MIN_LETTERS_NUMBER = 1
  LINE_LETTERS_NUMBER = 120
  AUTOCORRECT_DATA = {
    r: "are",
    u: "you",
    ASAP: "As soon as possible",
    ATB: "All the best"
  }

  # some methods
  # ...
end
~~~

While new features are being implemented, the number of static data increases and can cause havoc in the future. This code smell is called [Primitive Obsession](https://sourcemaking.com/refactoring/smells/primitive-obsession) and can be fixed by introducing Parameter (Value) Objects. To construct it you should look for logical links between primitive values and group them into the few immutable objects.

~~~ruby
class MailLengthSettings
  attr_reader :max_number, :min_number, :line_letters_number

  def initialize(max_number = 10_000, min_number = 1, line_letters_number = 120)
    @max_number = max_number
    @min_number = min_number
    @line_letters_number = line_letters_number
  end
end
~~~

~~~ruby
class Autocorrections
  DATA = {
    r: "are",
    u: "you",
    ASAP: "As soon as possible",
    ATB: "All the best"
  }

  def self.call
    DATA.all.collect do |abbr, transcript|
      new(abbr, transcript)
    end
  end

  attr_reader :abbreviation, :transcript

  def initialize(abbreviation, transcript)
    @abbreviation = abbreviation
    @transcript = transcript
  end
end
~~~

Here is an awesome [blog post](http://eftimov.net/primitive-obsession-ruby) with the detailed explanation.

## 2. Query Object

Query Objects are applied in case when you have the huge database query in controller:

~~~ruby
class PostsController < ApplicationController
  def index
    @posts = Post.includes(:author)
      .where(published: true)
      .where(author: { rating: 10 })
      .order("created_at DESC")
  end
end
~~~

I like to extract Query Object into modules that repeats class name of the model. The class name of Query Object explains the purpose of the request. In this case it would look like this:

~~~ruby
module Post
  class PublishedOfPopularAuthorsQuery
    attr_reader :relation
    private :relation

    def initialize(relation = Post.all)
      @relation = relation
    end

    def find
      relation.includes(:author)
        .where(published: true)
        .where(author: { rating: 10 })
        .order("created_at DESC")
    end
  end
end
~~~

And how a controller might use it:

~~~ruby
class PostsController < ApplicationController
  def index
    @posts = Post::PublishedOfPopularAuthorsQuery.new.find
  end
end
~~~

Here is a great [block post](http://craftingruby.com/posts/2015/06/29/query-objects-through-scopes.html) about this idea.

## 3. Service Object

It's a piece of the procedural programming style in Rails architecture. The class that commits only one thing. It's a pure ruby object, and you can create these classes by yourself. However, I prefer to implement services with a gem called [interactor](https://github.com/collectiveidea/interactor). Out of the box it provides such useful things as failed, successful context and also hooks. Sooner or later, you have to build these things yourself. For example, the service that created users from parsed CSV might look like this:

~~~ruby
class CreateUsersFromFile
  attr_reader :content
  private :content

  def initialize(content)
    @content = content
  end

  def call
    invalid_users = []

    ActiveRecord::Base.transaction do
      invalid_users = build_users.map.with_index do |user, index|
        { row: index, user: user } unless user.save
      end.compact

      raise ActiveRecord::Rollback if invalid_users.present?
    end

    invalid_users
  end

  private

  def build_users
    CSV.new(content, headers: true, header_converters: :symbol).to_a.map do |row|
      params = row.to_hash
      User.new(params)
    end
  end
end
~~~

Exracting this code from controller looks quite reasonable, doesn't it?

## 4. Decorator Object

It seems ideally suited to transfer logic out of models and helpers. The most trivial example would be the displaying user's `full_name`. From my point of view, [gem "drapper"](https://github.com/drapergem/draper) is the best choice to realize decoration logic. Before handing a model off to the view, you have to wrap it in a class that looks the following way:

~~~ruby
class UserDecorator < ApplicationDecorator
  delegate :id, :first_name, :last_name, :email

  def full_name
    "#{object.first_name} (#{object.last_name})"
  end
end
~~~

## 5. Form Object

This is a good spot to place a logic related only to the form. Therefore, it helps us to get rid of `accepts_nested_attributes_for`. To decouple your form object you can use [reform](https://github.com/apotonick/reform), [virtus](https://github.com/solnic/virtus) or implement [from scratch](https://www.reinteractive.net/posts/158-form-objects-in-rails). I prefer `reform`:

~~~ruby
class StoreForm < Reform::Form
  property :title, :city
  validates :title, presence: true

  property :coordinates do
    property :lat, :long
    validates :lat, :long, presence: true
  end

  collection :products do
    property :name
    property :quantity
  end
end
~~~

## 6. Policy Object

If you have roles in your app, from the several point you notice that authorization system is getting extremely complicated. Policy Objects provide you a simple, robust and scaleable way to polish user's responsibilities. I always use [pundit](https://github.com/elabs/pundit) for that purpose. It allows you to do the following:

~~~ruby
class PostPolicy
  attr_reader :user, :post

  def initialize(user, post)
    @user = user
    @post = post
  end

  def update?
    user.admin? || !post.published?
  end
end
~~~

For more detailed explanation follow the documentation.

## 7. Null Object

A [Null Object](https://robots.thoughtbot.com/rails-refactoring-example-introduce-null-object) is an object that helps us to avoid the presence checks. Here's the simple example from view:

~~~ruby
- if invitee.present?
  h2 = invitee.inviter
- else
  h2 No inviter so far
~~~

This is the stuff you definitely want to avoid. With the null object it can be converted to:

~~~ruby
h2 = invitee.inviter
~~~

To implement this we want to return an object that has an ability to respond to the given methods. For this type of objects I usually create a special folder `null` inside `models`. If you look inside, you’ll see classes like:

~~~ruby
class Null::Invitee
  def present?
    false
  end

  def inviter
    "No inviter so far"
  end
end
~~~

And again, I couldn't mention a brilliant [blog post](https://robots.thoughtbot.com/rails-refactoring-example-introduce-null-object).

## Conclusion

As you see there are many techniques to refactor a Rails app. Isn't this over-engineering? The answer to such a question is always context-dependent, but I rarely find that it is. Don't forget that code separation makes testing easier. If you stub many things in your tests it can be the indicator that you have to implement one of the aforementioned layers.

So, how many of them do you use? :smirk:
