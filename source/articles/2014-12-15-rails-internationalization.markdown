---
layout: post
title: "Rails Internationalization"
date: 2014-12-15
comments: true
category: rails
tags: rails, i18n
---
This post outlines an easy way to make Rails raise an exception when it finds
missing I18n keys. This is helpful in development and testing environments as a
way to quickly find and correct missing translation keys.

##Background
Ruby on Rails provides support for making your application work with different
languages. In other words, Rails provides a framework for making sure that
things like column names and other bits of natural language can be translated
between many different languages. This support is provided by the I18n gem,
which is included with Rails. The Rails Guide for this is 
[here](http://guides.rubyonrails.org/i18n.html).

##Keys
There are many ways to implement I18n support in your application. A common way
is to use the #translate or #t helper method provided by Rails. This method
takes a key as an argument and returns the string specified in the appropriate
yaml file. You can also call I18n.translate directly.

    t(".my_key")

or 

    I18n.translate(".my_key")

##Problem
The only difference between calling translate() (or t()) and I18n.t() is the latter
will rescue any exceptions
that arise from missing keys, and return a formatting html string with a
humanized version of the key. In other words, if Rails cannot find a key
".default_text", then it would return "Default Text", and that's what users would
see.

This behavior is desirable in production, where you don't want your app to crash
because of a missing translation. But not in development or testing
environments, where you'd like to be able to easily catch missing translation
keys.


##Solution
If you happen to be calling I18n.translate directly, or for some other reason
are not using the Rails-provided helper methods, then this 
[blog post](http://robots.thoughtbot.com/foolproof-i18n-setup-in-rails) from
[Thoughtbot](http://thoughtbot.com/) provides some useful information about
custom exception handlers for I18n.

The problem with the above approach when using the Rails #translate (and #t)
helper methods is that no exceptions get raised, by default. (At least that was 
the case for me, and I was forced to figure out another solution.)

The solution is to explicitly enable this behavior by passing the option 
raise: true to the helper method:

    t(".some_key", raise: true)

This works, but it requires a lot of repetition and is hard to apply to an
existing code base. To automate this, you can create an initializer that inserts
the raise: true option into all the #translate method calls, but only in certain
environments:

    if Rails.env.development? || Rails.env.test?
      module ApplicationHelper

        def translate(key, options = {})
          super(key, { raise: true }.merge(options))
        end

        alias :t :translate
      end
    end




