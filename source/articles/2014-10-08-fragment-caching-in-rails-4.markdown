---
layout: post
title: "Fragment caching in Rails 4"
date: 2014-10-08
comments: true
category: rails
tags: rails
---

Recently I implemented a nested caching scheme in a Rails application. I ran
into a couple of interesting problems and found the documentation confusing, so
I thought I'd write up what I did. This post is aimed at somebody new to 
caching. 

What is caching?
----------------
In this post I'm referring to caching as pre-generating chunks of html. Not 
caching database queries, pages, controller actions, etc. 

Why?
----
Because it's fast. If you watch your server log, you'll see that rendering the
views takes quite a bit of time. At least for me, this was usually 10x or 20x 
the database query time. When you cache a part of a view, Rails does not have to
generate the html. This saves a ton of time. 

How?
----
If you look at the 
[Rails guide page on caching](http://guides.rubyonrails.org/caching_with_rails.html)
you'll see a bunch of stuff in there about page and action caching, along with 
a helpful note those things are deprecated in Rails 4. Also a link to a blog 
post by DHH (the "founder" of Rails) with an explanation of something called
key-based caching. 

At first I found this all very confusing, but it can be really simple. The "chunks"
of pre-generated (cached) html are stored in an in-memory key-value store. The
value is the html. The key is created by you. 

First, we need to worry about where we'll store these keys and values. Locally,
this is easy to set up based on the instructions in the Rails guide above. 
Production varies of course. Heroku has add-ons, I used 
[Memecachier](https://www.memcachier.com/). Works well for me. There is good 
documentation about this, so I'll ignore the setup details and focus on the code.

To cache a part of a view, wrap that code in a cache block:
    
    <% cache(my_cache_key) do %>
      #cached erb, hml, html etc here
    <% end %>

The part to pay attention to is the keys. You want to create a key that will 
change when you want rails to re-generate the view code, but not change 
otherwise. Rails will look for a value for that key when a user asks for a page,
if the key doesn't exist that will force Rails to re-generate that view. If the
key is there, Rails will just served the cached copy. 

A simple example would be for some static part of your page, perhaps
a footer:

    <% cache(["v1-footer") do $>
      # some html 
    <% end %>

If you updated your footer view partial, then you would simply change the "v1"
string to "v2-footer". The next time Rails renders the 
footer, it will look for a cache key "v2-footer". It won't find one, so it will
render the footer and make a new cache entry. From then on, Rails will find that
key and all will be well. 

You can take this simple idea and expand it to dynamic content. Let's say you 
have a view listing all the products a company sells. You want to cache this
view, but if you take the approach above, it won't be updated when a product 
changes. The solution is to create a cache key that changes when any product
changes, *and* when you want to manually change it (for example if you change
the view code). 

    <% cache ["v1", cache_key_for_products] do %>
      # Your index view here
    <% end %>

cache_key_for_products is simply a helper method that returns a string that
changes whenever any product is updated (this is straight from the Rails guide:

    module ProductsHelper
      def cache_key_for_products
        count          = Product.count
        max_updated_at = Product.maximum(:updated_at).try(:utc).try(:to_s, :number)
        "products/all-#{count}-#{max_updated_at}"
      end
    end

Rails will combine the "v1" with a string that will change when the a product is
updated. So the cache will be bust in the case of product change, and also when
you change the "v1" bit, probably because you updated the view code and you need
to re-generate all the cached html. 

Ok, so we're pretty far along already. The final step is to set up a nested 
cache scheme. Why should we re-generate html for each product, when only one
has changed? Let's cache each table row, too. 

    <% cache["v1", product] do %>
      <tr>Great product</tr>
    <% end %>

This will generate a cache entry with a key that combines the template version
("v1") with a fingerprint of the product record. By wrapping each row in a cache
block, then caching the whole table, we have a slick nested caching setup that
will minimized the time spent rendering views. Now, when one product is updated,
Rails simply needs to regenerate 1 table row, reassembly the already-cached other
rows, and we're all set. This is a lot faster than doing it all from scratch
every time. 

So that's about it, as far as basics go. One note -- there are other ways to do 
this. In this setup I describe, all cache keys are universal. I find that easier
to deal with, but you can also use the action or controller context to define 
keys. This is covered in the Rails guide. 

Gotchas
-------

What about paginated index views? If you're using will_paginate, this should
work:

    <% cache ["v1-#{params[:page]", cache_key_for_products] do %>
      # Your index view here
    <% end %>

What about a paginated index view that also has a Ransack search form? 

    <% cache ["v1-#{params[:page]-#{params[:q]", cache_key_for_products] do %>
      # Your index view here
    <% end %>



