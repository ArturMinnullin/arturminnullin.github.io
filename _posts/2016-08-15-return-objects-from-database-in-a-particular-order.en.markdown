---
layout: post
title: Return records from DB in a particular order
description: Return records from DB in a particular order using ActiveRecord
date: 2016-08-15 00:00:00
---

Let's imagine that somehow we have an array of ids in a specific order. We can obtain them from the from on the view or as results of complicated SQL. And now we want to get records in the order that array dictates. The usual manner doesn't fit...

~~~ruby
Product.find([1, 25, 12])
~~~

...because it generates the simpest SQL that uses index search...

~~~bash
Product.find([1, 25, 12]).to_query
Product Load (0.5ms)  SELECT "products".* FROM "products" WHERE "products"."id" IN (1, 25, 12)
~~~

... and return records in order that is typical for index (BTree in this example). Therefore, the pure ActiveRecord is not sufficient here. We need to fix this by taking advantage of SQL magic :sparkles:

So one way how to deal with this is to define the method that "rank" products, use the resulting structure for inner join and order them by rank. Let me demonstrate what I mean:

~~~ruby
def ranked_products
  products.map.with_index do |product, index|
    "(#{product.id}, #{index})"
  end.join(', ')
end

Product
  .joins("INNER JOIN (VALUES #{ranked_products}) AS ids(id, rank) ON products.id = ids.id")
  .order("ids.rank DESC")
~~~
Notice that in the above code the format of `"(#{product.id}, #{index})"` is related to the requirements that PostgreSQL has.

I've provided a trivial example here when the rank method uses the index number. However, there are different variants are possible.

I hope you found this helpful!
