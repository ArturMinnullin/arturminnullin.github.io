---
layout: post
title: How many types of objects your Rails app uses?
---

Ok guys, I know that you work or worked with legacy code. And I pretty sure that you are faced with dread and horror.

The main issue is that standard MVC stack is not enough in many cases. Models and controllers start to grew and violate Single Responsibility Principle.

Don't talk about specific OOP pattern. It's new layers for your rails application.

## 1. Parameter object

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

While new features are implementing the number of static data increase and can cause havoc in the future. This code smell is called Primitive Obsession and can be fixed by introducing parameter (value) objects. To construct it you should look for logical links between primitive values and group them in the few immutable objects.

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

Here is awesome [blog post](http://eftimov.net/primitive-obsession-ruby) with detailed explanation.

## 2. Query object

Query objects are applied when you have the huge database query in contoller.

## 3. Service object

It's a piece of the functional programming style in Rails OOP architecture. The class that commit only one thing.

## 4. Decorator object

They seems ideally suited to tranfer logic from models. The most ptimitive exx

## 5. Form object

## 6. Policy object

## 7. Null object
