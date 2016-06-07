---
layout: post
title: Return records from DB in a particular order
description: Return records from DB in a particular order using ActiveRecord
date: 2015-07-05 00:00:00
---

Let's imagine that somehow we obtain array of ids in a specific order.

def ranked_products
  @ranked_products ||= FilterProductsByEffects.new(effects).call.map do |product|
    "(#{product.id}, #{rank(product.effects.pluck(:kind))})"
  end
end

Product
  .joins("INNER JOIN (VALUES #{ranked_products.join(', ')}) AS ids(id, rank) ON products.id = ids.id")
  .order("ids.rank DESC")
