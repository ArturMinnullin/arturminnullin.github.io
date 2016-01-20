---
layout: post
title: Image generation with RMagick and Single Responsibility Principle
description: Use rmagick in order to generate image with many layers
date: 2015-07-31 00:00:00
---
Recently in one of our projects I was confronted with a feature related to sharing on Facebook. Facebook has an extremely simple API for this very purposes. It requires only a single parameter - API key. However, you can customize the text and the image in the Share dialog and Post by changing these values: title, description and image url.

In my case the issue was about the image not being static and the necessity of it being randomly generated every time we refresh the page. The pattern for each image includes:

1. A background image as a basis.
2. A random title taken from the record of certain model.
3. Logo of the web-service.


To achieve the goal I decided to use `gem "rmagick"`. RMagick is an interface between the Ruby programming language and the ImageMagick and GraphicsMagick image processing libraries. It is a great tool in case you need to manipulate graphics from the code side. We are going to take an advantage of some cool features implemented in this gem.


So, let me get straight to the point. First of all, I have added `gem "rmagick"` to my Gemfile. Then, told Rails to install the new gem:

```ruby
bundle install
```

Let's analyze what we're going to do before jumping into the code. Basically, we have two options for drawing something on the image: place some text on canvas and impose images surfaces. At first sight, the best solution is to construct a service that contains two actions for each mode of drawing. My new class is being instantiated in the following way:

```ruby
class WatermarkedBackground
  BACKGROUND_PATH = "/path/to/background.jpg"

  attr_reader :text, :annotation, :logo
  private :text, :annotation, :logo

  def initialize(text, logo_path)
    @text = text
    @annotation = Magick::Draw.new.tap do |text|
      text.font = "Helvetica"
      text.pointsize = 60
      text.font_weight = Magick::BoldWeight
      text.fill = "black"
      text.gravity = Magick::CenterGravity
    end

    @logo = Magick::Image.read(logo_path).first
  end

  def perform
    draw_text
    append_picture
    save_image
  end

  private

  def draw_text
    annotation.annotate(background, 0, 0, 0, 60, text)
  end

  def append_picture
    background.composite!(logo, 420, 20, Magick::OverCompositeOp)
  end

  def save_image
    background.write("/path/to/generated_image.jpg")
  end

  def background
    @background ||= Magick::Image.read(BACKGROUND_PATH).first
  end
end
```

Let's take a look at what we've done here.


The first action `draw_text` prints our random content in the center of 'background.jpg'. The base player of this action is `annotate` function. It takes six arguments:

* the image you want to annotate
* the width and height of the rectangle for the positioned text
* the two arguments are offsets for x, y axes respectively
* the last one is the text for the annotation

I configured width and height are equal to zero. It indicates that the whole image should be used as an annotation area. The attribute `gravity` points out the position of text inside rectangle. In my case it is the center of rectangle.
Also, I specify the font, the text color, size, and other properties of the text in `initialize` method.


The second action `append_picture` gives us the ability to blend two pictures into a single image. The main part here is `composite!` function. To push it into the work we should define four arguments. Three of them are pretty obvious: source image, x- and y- offsets relative to the upper-left corner of the background image. The last argument is a little bit tricky. It is associated with algorithm used to compose the first image with the second. The number of options is about 60 variants. You can review here [documentation](http://rmagick.github.io/constants.html#CompositeOperator) to figure out which one fits you best.


The last action `save_image` saves prepended image and automatically converts it to the extension that we specify. The call of `WatermarkedBackground.new("The quick brown fox jumps over a lazy dog", "/path/to/logo.png").perform` gives me the desired result.


After I have dotted the i's and crossed the t's it's time to ask: Is this the best way to organize the code? Unfortunately, not. Now I have single class that contains the both behavior and configurations for the image generation. However, a class should do the smallest possible useful thing; that is, it must not violate a single responsibility principle. It would be difficult to test actions of my class in isolation of the text properties. I'd have to stub too many methods.


So it makes sense to separate the configurations for the annotation text in the independent class that is wrapped the RMagick standard action `annotate`, as shown bellow:

```ruby
class TextImage
  attr_reader :text
  private :text

  def initialize(text)
    @text = text
  end

  def annotate(*args)
    draw.annotate(*args, text)
  end

  private

  def draw
    Magick::Draw.new.tap do |text|
      text.font = "Helvetica"
      text.pointsize = 60
      text.font_weight = Magick::BoldWeight
      text.fill = "black"
      text.gravity = Magick::CenterGravity
    end
  end
end
```

And class can be rewritten in the following way.

```ruby
class WatermarkedBackground
  BACKGROUND_PATH = "/path/to/background.jpg"

  attr_reader :logo, :text, :background
  private :logo, :text, :background

  delegate :write, to: :image

  def initialize(text:, logo_path:)
    @text = text
    @logo = Magick::Image.read(logo_path).first
    @background = Magick::Image.read(BACKGROUND_PATH).first
  end

  def image
    @image ||= text.annotate(composited_background, 0, 0, 0, 0)
  end

  private

  def composited_background
     background.composite(logo, 420, 20, Magick::OverCompositeOp)
  end
end
```

As you see I decoupled my code and reduce dependencies. This polishing allows me to remove the responsibility for some of RMagick methods. No need to stub them for the testing `WatermarkedBackground` anymore. It is substituted by only one responsibility `TextImage` which can be easily reused without duplication.

```ruby
WatermarkedBackground.new(
  text: TextImage.new("The quick brown fox jumps over a lazy dog"),
  logo_path: "/path/to/logo.png"
).write("/path/to/generated_image.jpg")
```

Check out the result of my work below.
![Result]({{ site.url }}/assets/generated_image.png)
Nevertheless, RMagick has a bunch of cool features. They can be found at their [official web-site](https://rmagick.github.io/).


P.S. Special thanks go to [Vladimir Mikhailov](https://github.com/VladimirMikhailov) for taking the time to review this post and give terrific advice that lead to the great improvement.
