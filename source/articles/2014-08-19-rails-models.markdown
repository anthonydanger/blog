---
layout: post
title: "ActiveRecord Models"
date: 2014-08-19
comments: true
category: rails,
tags: rails
---

This post is a writeup of a [RailsSchool](http://www.railsschool.org/) 
[class](http://www.railsschool.org/l/intro-to-rails-models) on 14-08-19 on 
ActiveRecord models. 

What is a model?
----------------
In web programming, model generally refers to the “M” in MVC. The model’s job is 
to be an interface to your database. More precisely, the model is a programmatic 
representation of your data structure. This is called an ORM, or 
“object-relational mapping.” Sometimes models include basic logic relating to 
data, such as validations. For example, you might decide that an employee’s 
hourly rate should never be below minimum wage. That logic is something that 
would usually go in a model. 

In Ruby on Rails in particular, models are classes that descend from 
ActiveRecord::Base. Instances of these classes represent a single data entity. 
For example, if you are using a traditional SQL database such as PostgreSQL, 
each model will probably represent a specific table. Each instance of the model 
class will represent one row in the table. 

ActiveRecord models are defined like so:

    
     class Customer < ActiveRecord::Base
        # some other stuff goes here     
     end

You can create an instance of one of these classes just like any other class:

     new_customer = Customer.new

ActiveRecord models create instances where methods and attributes correspond to 
database columns. So something like

     new_customer.first_name = “Ward”
     
will set the first_name attribute of the new_customer object to “Ward”. In most 
cases this will make it into the database.

Using ActiveRecord models
-------------------------
ActiveRecord supplies us with all kinds of useful methods. Let’s explore some of 
the most commonly used. This section should give you a basic sense of how to use 
ActiveRecord in your Rails projects. For more information, see this incredibly 
[useful guide](http://guides.rubyonrails.org/index.html).

###Migrations

ActiveRecord is an interface to a database, to to use it you will probably first 
create a database and use a migration to add tables, columns, or otherwise 
modify your data structure. Migrations are a topic themselves, the 
[RailsGuide](http://guides.rubyonrails.org/index.html) has some more
information.

###Class methods
####Making new objects
    ::new => creates a new object, but does not save it in the database
    ::create => creates a new object, AND saves it in the database
    ::create! => same as above, but raises an exception if the object is not saved

####Finding existing objects  
    ::find => takes an id, returns an object
    ::where => takes a hash, returns a collection
    ::first => returns the first object
    ::second => returns the second object
    ::last => returns the last object
    ::all => returns all the objects
    
####Destruction
    ::destroy_all => all objects are destroyed by calling #destroy
    ::delete_all => objects simply removed from the db. skips callbacks.
     
###Instance methods
    #save => save the object to the database
    #update_attributes => takes an attributes hash as an argument, updates the attributes, and saves the record
    #destroy => removes the object from the database, and run callbacks
    #delete => just removes the object from the database

In addition to a variety of instance methods, ActiveRecords also gives us 
instance methods that correspond to data object attributes (generally columns). 
Things like #first_name, etc. Also, we get instance methods that are mapped to 
object relations. For example, some_customer.orders, or subscriber.invoices. 
These relation methods can be used to quickly return collections of related data 
objects.

Creating ActiveRecord models
----------------------------
ActiveRecord is a large and powerful library. This section is an overview of the 
most comment ActiveRecord features you will use while working with 
Ruby on Rails. 

###Naming conventions
ActiveRecord models are named according to convention, where the class name is 
the singular version of the table name. For example, an AR class called “User” 
would correspond to a database table named "users".

####Reserved words
There are some words you don’t want to use, because they’ll conflict with other 
parts of Rails or Ruby. Class names like Session, Database, Application, would 
be bad. Also, you don’t want to name your attributes things like “transaction” 
or “relationship".

###Relationships

Standard SQL relationships are set up using keywords in the ActiveModel class 
declaration. Some examples are:

     class Address < ActiveRecord::Base
          belongs_to :customer
     end

     class Customer < ActiveRecord::Base
          has_many :orders
     end

Notice that the declarations use Ruby symbols as arguments, and the symbols 
correspond to other ActiveRecord classes. Notice also that the first 
relationship uses a singular name, while the second uses a plural, as you would 
expect. You can find much more information in the RailsGuide section on 
ActiveRecord relationships. Of particular note is 
the dependent: :destroy argument:

     class Customer < ActiveRecord::Base
          has_many :orders, dependent: :destroy
     end

This tells Rails that when a Customer object is destroyed, you also want all the dependent Orders to be destroyed as well. 

