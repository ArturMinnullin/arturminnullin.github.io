---
layout: post
title: Saving images from base64 format
description: Saving images from the Ruby on Rails API in base64 format
date: 2015-11-19 00:00:00
---

This post explores methods of saving an image through the API. I have experimented with three different image processing gems in order to cover as many cases as possible.

First of all, let's see the entirety of the situaltion. There are two different applications:

* A frontend application. It has the "Choose file" field on the particular form
* A backend API application that receives a request after submitting the form

The question is how to save the uploaded image? I decided to use the following method. I convert the image to base64 scheme, send it, then decode it on API-side and save as regular image.

Here's how my es6 (frontend) code looks like. Nothing fancy here, I just trigger `change` event on the file field and apply `FileReader` interface to read the contents of it. I use a hidden field to save decoded image but there is not a problem to bring some javascript framework into the play.

~~~javascript
$("#attachment_image").on("change", (event) => {
  const reader = new FileReader();

  reader.onloadend = () => {
    $("#attachment_image_coded").val(reader.result); // save encoded image
  }

  reader.readAsDataURL($(event.currentTarget)[0].files[0]);
});
~~~

At the moment the backend accepts a signature image in base64 format and I have to supply incoming requests. Let me show you how to use three different adaptors to decode the `image` and save it to my model that's called `product`.

## Paperclip

Paperclip (>= 4.0) automatically detects and supports base64 data with `io_adapters` method. It acts on any field matching the base64 regex.

~~~ruby
Product.new.tap do |product|
  product.logo = Paperclip.io_adapters.for(params[:image_coded])

  product.save
end
~~~

## Carrierwave

Simply use [carrierwave-base64](https://github.com/lebedev-yury/carrierwave-base64) gem. You need only two lines to roll it out. In Gemfile:

~~~ruby
gem "carrierwave-base64"
~~~

And mount the uploader to model:

~~~ruby
mount_base64_uploader :image, ImageUploader
~~~
Then implement behavior following the standard flow.

## Refile

With Refile things are getting a little bit complicating. The issue is, `refile` doesn't have base64 encoding out-of-the-box. So I need to decode image manually and create the temporary file. The string with the encoded data should not be prefixed with Data URI scheme format, therefore I use `partition` to get the substring. The script I ran along:

~~~ruby
decoded_image = Base64.decode64(params[:image_coded].partition(";base64,").last)

Tempfile.new("temp_file").tap do |file|
  file.binmode
  file.write(decoded_image)
  file.close
  product.image = file
  product.save
end
~~~
As you can see, `carrierwave` and `paperclip` are more convenient and user-friendly, but if you're looking for a deeper understanding of the whole image upload process, you should definitely check `refile` out.

And we're done. Thank you for reading!
