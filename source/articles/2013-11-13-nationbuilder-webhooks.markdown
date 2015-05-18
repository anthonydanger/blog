---
layout: post
title: "NationBuilder Webhooks"
date: 2013-11-13 20:42
comments: true
category: Rails
tags: webhooks, rails, api, nation_builder
---
I recently built a Rails application that integrates with NationBuilder
and thought I'd share the details. NationBuilder has two APIs. One is a
RESTful API, and the other wis a webhooks API that posts JSON data when
certain things happen on a NationBuilder site. This post is concerned
with the latter. The goal is to build a list of donations that occurred
on a NationBuilder site over a period of time. This is not possible with
the RESTful API (I guess it's not fully RESTful?). So instead I'm
building a controller to "listen" for the webhooks payloads and storing
them. 

The first step was to set up a nation to post data. I already have
access to a nation in production, so I have plenty of live data to deal
with. I assume if you're reading this you can use The Google to figure
out how to do this part. You should set up an additional webhook to post
stuff to requestb.in (or something similar) so you can see what is going
on. 

Next set up a route. Obviously this route must match where you told
NationBuilder to post to. 

    match '/webhooks/nb/create', to: 'nationbuilder#create', via: :post

This will take anything being posted to myapp.com/webhooks/nb/create and
rout it to the create action of my nationbuilder controller. 

So on to the controller.

    class NationbuilderController < ApplicationController
      skip_before_filter
      protect_from_forgery :except => :create

      def create
        render nothing: true
        # do stuff
      end
    end

All I want to do is grab the json and store it to my database using
ActiveRecord. So I decided to use the create action, though I 
think #new would probably be fine, too. I've omitted the code that 
actually stores the data for clarity. 

There are three things to get right, and then it works like magic:
1) If you have any before filters at all, you probably need to skip
them. EDIT: The method above doesn't seem to work with Devise. Instead,
have your NationBuilder controller inherit direction from 
ActionController::Base. 
2) protect_from_forgery turns off a default security feature. You want
to do this -- the security feature is meant to prevent people posting
random data to your controller. Since that's exactly what NationBuilder
will be doing we need to allow this. Notice it's limited to the only
action we'll be defining. If you are going to be dumping data from the
JSON payload into a database you should probably be concerned about SQL
injection. 
3) Your controller methods need to be told not to render a template.

Testing is another story. I'd like to test this with an integration
test, but I'm having a hard time simulating a JSON post. StackOverflow
is full of info about this but nothing works.  NationBuilder,
fortunately, lets you send test payloads with a click. So for now I'm
testing the code that way. I'll update this when I figure out the
integration tests. 

