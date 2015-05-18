---
title: "ActBlue API"
date: 2013-11-04 21:50
tags: Getting Started
layout: post
tags: rails, act_blue, api
---

I recently set up a Rails app to use the [ActBlue XML API](https://secure.actblue.com/api). The goal was to use
their API to pull down contribution information across campaigns. Here
is what I learned:

The most important thing I learned is that the ActBlue API is no longer
supported. At least, that's what an ActBlue staffer told my client when
we informed them of an issue with their API. The API currently works,
and they don't seem to have plans to turn it off. I'm a bit annoyed
ActBlue didn't mention this on their web site. As I'm working on this
particular project I've found that APIs offered by companies who's
primary service is something else are often really spotty. 

The first task was to write a simple Ruby wrapper. There's [a gem](https://github.com/netroots/ruby-actblue),
but it's old, doesn't quite fit my case, and seems overly complex for my
needs. So I wrote a really simple wrapper:

    ACTBLUE_URI = "https://secure.actblue.com/api/2009-08"
    HEADER = { 'Accept' =>'application/xml' }

    class Campaign

      attr_reader :entity

      def initialize(username, password, entity)
        @auth = { username: username, password: password }
        @entity = entity
      end

      def details
        HTTParty.get("#{ACTBLUE_URI}/entities/#{@entity.to_s}", basic_auth: @auth, headers: HEADER)
      end

      def all_contributions
        HTTParty.get("#{ACTBLUE_URI}/contributions?destination=#{@entity.to_s}", basic_auth: @auth, headers: HEADER)
      end

      # method arguments must be ISO 8601 timestamps (ie Time.now.iso8601)
      def contributions_in_date_range(start_time = '', end_time = '')
        HTTParty.get("#{ACTBLUE_URI}/contributions?destination=#{@entity.to_s}&" +
                     "payment_timestamp=#{start_time.to_s}/#{end_time.to_s}",
                     basic_auth: @auth, headers: HEADER)
      end
    end


The last method is the one I really use; it returns an
HTTParty::Response object, which I convert to a hash by calling
the parsed_response method. Unfortunately, the structure of the hash is
different for results with 1 or many contributions. For now I'm pretty
much manually mapping the hash fields to my ActiveRecord object, and I
have to do it differently for cases where there is only 1 response.
Annoying, but I'm waiting for the project to move on before I try to
abstract the whole process to something more elegant. 

You can see from my comments the timestamps must be in iso8601 format.
This is in the docs. But what *is not* in the docs is ActBlue will
return a timestamp not valid error if you pass UTC times. In other
words, if your UTC offset is +0, the API will fail. So I set Heroku to
work on local time, and all is well. A better approach is probably to
convert the timestamps to localtime using Ruby, instead of relying on
the server's configuration, but this works fine for now.

I final thing I learned, also very odd: ActBlue will give you the
campaign details for *any* campaign, even if your user doesn't have any
permissions associated with that campaign. I suppose this is because the
details are public anyway. But it can cause problems if you, like I did,
make a quick call to the details API method to verify your API
credentials. It will work even if you don't have any real permissions
for that campaign. 
