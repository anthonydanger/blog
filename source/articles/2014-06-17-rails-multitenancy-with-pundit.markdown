---
layout: post
title: "Rails Multitenancy with Pundit"
date: 2014-06-17 12:16:59 -0700
comments: true
category: Rails
tags: rails, pundit
---
I just finished building my first multi-tenant app with Rails. I used Pundit's policy scopes to handle scoping the index
views, and used a many-to-many relationship between users and my tenant model. Below is a description of how I set
everything up.

##Problem
My app currently has these (and more) models: User, Company, Customer, Payment. The relationships were straightforward:
Users belonged to a Company, which had many Customers. A customer had many Payments. The app is primarily an accounting
application, and using it consisted of admin-level users logging in and performing the same set of tasks for each Company. This
is the desired workflow. My task was to allow this workflow to continue, but to allow each Company's users to log in and
access most of those same tasks.

While researching this, I concluded there were two main approaches to handling multitenancy with Rails: 
*Use scopes, as described by [Ryan Bates](http://railscasts.com/episodes/388-multitenancy-with-scopes)
*Use database schemas, like the [Apartment gem](https://github.com/influitive/apartment)

##Solution
My first solution was to use scopes in the controller like Ryan Bates does, combined with namespacing controllers to
handle some of the features that only admin users could access. This proved to be overly complex and lead to lots of
repeated code. So I refactored the app along the following lines, and I am pleased with the result. 

###Tenant model
The first step is to decide what tenant model the application will use. In my application the Company model was the
clear choice. You may have to create a Tenant model. 

###Users-tenants
My app has several views that should only show records which the current user can access. For example, the
customer#index action should only return Customers that are related to a Company which is related to the current user.
The first step to accomplishing this is to define the relationship between Users and your tenant model, Company in my
case. The easiest way to do this is to have a User belong to a Company, which can have many Users. This isn't ideal, 
however, because you probably have some users that should be able to access many Companies. In my case all my app's
existing users needed to access all the existing Companies. In this case a many-to-many relationship is the right
way to go. 

In Rails (4, at least) there are two main ways to accomplish this: has_and_belongs_to_many, and has_many: :through. I 
went with the latter method after the former would not work for me. The first step was to remove the company_id field
from my User model. After that, I created the following migration

    class CreateCompanyUsers < ActiveRecord::Migration
      def change
        create_table :company_users do |t|
          t.belongs_to :compnay
          t.belongs_to :user
          t.timestamps
        end
        add_index :company_users, [:company_id, :user_id], unique: true
      end
    end

Then a simple model file

    class CompanyUser < ActiveRecord::Base
      belongs_to :compnay
      belongs_to :user
    end
    
    
In the User model, I added these two lines

    has_many :company_users
    has_many :companies, through: :company_users
    
In the Company model

    has_many :company_users
    has_many :users, through: :company_users
    

These changes are sufficient to set up a many-to-many relationship, so that a User can be associated with many Companies
and vise versa. 

###User roles
I had a previously set up a roles system for Users. In config/application.rb, I have this

    config.user_roles = %w[staff manager admin super_admin]
    
My User model has a role column which stores a string, and I have the following in the Role model file

    def role?(base_role)
      role.present? && User.roles.index(base_role.to_s) <= User.roles.index(role)
    end
  
    private
      def self.roles
        Lupine::Application.config.user_roles
      end
      
This allows me to do things like this in controllers or policies

    current_user.role? :admin
    
That method will return true if the current user's role is 'admin' or 'super_admin', and false if the role is '', 
'staff', or 'manager'

###Authorization
I've been using [Pundit](https://github.com/elabs/pundit) for authorization, and really like it. Pundit has a handy
solution to the problem of authorization scopes - providing records to which the current user has access. The writeup
in their docs is good, and I won't duplicate it here. 

In order to use policy scopes you need to have a class method that takes a user as an argument and returns a collection
of authorized records. There might be other ways to make it work, but this is what I did and is the most obvious approach.
I have already detailed the relationship between Users and the tenant model. To make this work, you also need a relationship
between the tenant model and all the child models. In my app that meant I needed to create a direct one-to-many 
relationship between Company and Payment. In addition to that change, I did a few more things.

First I changed the Company, Customer, and Payment models to have a users relationship/method. This was easy to 
accomplish using has_many :users, though: :company  since both models relate to Company, the tenant model. 
      

Then I created a concern with the following method

    module ModelScopes
      def user_records(user)
        select { |record| record.users.include?(user) }
      end
    end
    
I extended Company, Customer, and Payment with this module. Now I had a class method, ::user_records, that would return
the correct records. I could use this method with Pundit's policy scopes like this

      class Scope < Struct.new(:user, :scope)
        def resolve
          scope.user_records(user)
        end
      end

This worked well, except for one problem. Using ::select returns an Array, not an ActiveRecord::Relation. Pundit expects
the latter and uses that to decide what policy class to use. Fixing this turned out to be fairly easy. I simply
wrote my own authorize method. My controllers now look like this, and things are working nicely. 
    class CompaniesController < ApplicationController
    
      include Pundit
      after_filter :verify_policy_scoped, only: :index
      after_filter :verify_authorized, except: :index
      
      def index
        @campaigns = policy_scope(Campaign)
        raise Pundit::NotAuthorized unless current_user.role? :staff
      end
   
    end

When a user visits
the Company index view, she sees all the companies to which she is related. I can take advantage of the current_user 
method to restrict this action based on a user's role, as well. This gives me a fairly good degree of flexibility, is 
simple, and keeps things nice and separated.

One drawback is that I cannot test my my policies for correct behavior on the index action. I'll cover this area of the
application with a model-level test to ensure the correct records are being returned, and with an feature-level spec
to make sure it all works together. 


