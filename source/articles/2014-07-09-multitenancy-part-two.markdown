---
layout: post
title: "Multitenancy Part Two"
date: 2014-07-09
comments: true
categories: [Rails, Pundit]
---
In my [previous post](/blog/2014/06/17/rails-multitenancy-with-pundit/) I wrote 
about who to create a multi-tenant Rails application using the Pundit gem.
 
I've been living with this design for a month or so now and it's worked well so
far. One case where it does not work well is when I want to have a 
parameterized controller method. For example, let's say I have an application
with a Supplier and Product model, and a Supplier has many Products. 

Sometimes I want my products#index view to show all the products across
suppliers, sometimes I want it to show only the products for a specific
supplier. 

Normally this is very easy to accomplish with something like this

    class ProductsController < ApplicationController

      def index
        if params[:supplier_id]
          @products = Supplier.find(params[:supplier_id].products
        else
          @products = Product.all
        end
      end
    end

However since I'm using Pundit I can't easily do that, because my index actions
receive a collection of objects from the policy scope method. The latter passes
a method to the class in question. Below is what I did to solve this problem. In
this case I'm always expecting a supplier_id, but you could easily use an if
statement to check for it. 
I should probably refactor this, but I wanted to note it for future reference.

    class ProductsController < ApplicationController

      def index
        scope = policy_scope(Product)
        @products = scope.select do |product|
          product.supplier_id == params[:supplier_id].to_i
        end
      end
    end

