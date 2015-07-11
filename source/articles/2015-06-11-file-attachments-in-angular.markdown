---
layout: post
title: "File Attachments in Angular with Rails, S3, and Paperclip"
date: 2015-06-11
comments: false
tags: angular, rails, paperclip, aws, s3
---

Web applications commonly require file attachments. This posts walks
through an approach to setting up secure file attachments in a web application
using [AngularJS](https://angularjs.org),
[Ruby on Rails](http://rubyonrails.org),
[Paperclip](https://github.com/thoughtbot/paperclip),
and Amazon's [S3](https://aws.amazon.com/s3).

##AWS
The first step is to create an account with [AWS](https://aws.amazon.com).

###S3
Click on the S3 menu on the AWS web console, and create a bucket. I usually
make different buckets for different environments: production, staging,
development, and use the development bucket for testing. I also generally
make different buckets for different types of files -- for example,
file attachments, archived reports, database dumps, etc.

I might have the following buckets:

my-app-dbbackups  
my-app-attachments  
my-app-quarterly-reports  
my-app-log-archives  

Paperclip will namespace your attachments by resource type, so if you have a
few different models that all have attachments they'll be nicely organized.

###IAM
Now we need to set up an AWS user for our application.  It
is a good idea to create different users for different applications and to
limit the user's permissions. To do this, lick on "Identity & Access Management"
in the AWS web console. Then click on "Users" on the left. Click the
button to create a new user, and be sure to download the security credentials.
We'll use them later.

You might want to use Groups to manage your application users, but we'll skip
that for now. We simply need to create a user policy that allows access to the
appropriate buckets. Create a custom policy using this template, where
my-bucket is the name of your bucket. Repeat the pattern for multiple buckets.

```json
{
    "Statement": [
        {
            "Action": "s3:*",
            "Effect": "Allow",
            "Resource": [
                "arn:aws:s3:::my-bucket",
                "arn:aws:s3:::my-bucket/*",
                "arn:aws:s3:::my-bucket2",
                "arn:aws:s3:::my-bucket2/*"
            ]
        }
    ]
}
```

##Rails
Now we'll do some basic setup of our rails application.

###AWS Credentials
Since we just set up an AWS user for our rails app, let's add those credentials
to the app. You can do this anyway you like, but I prefer either the [Figaro
gem](https://github.com/laserlemon/figaro) or using the [config/secrets.yml
pattern](http://guides.rubyonrails.org/4_1_release_notes.html)
implemented in Rails 4.1 or later. For this article I'll use config/secrets.yml.
Here is an example secrets.yml. Notice the production section references ENV
variables. This pattern lets you set these secrets in the repo for development,
but on Heroku you can use the standard environment variables.

```yaml
development:
  aws_key_id: my_key
  aws_secret: my_secret

test:
  aws_key_id: my_key
  aws_secret: my_secret

production:
  aws_key_id: <%= ENV['AWS_KEY_ID'] %>
  aws_secret: <%= ENV['AWS_SECRET'] %>
```

###Paperclip
The [Paperclip docs](https://github.com/thoughtbot/paperclip) should give you
all the information you need. Follow their instructions to set things up. In
particular you should pay attention to the Security Validations section to
prevent spoofing of upload file type. Another aspect of
security - making sure your files are only accessible to the correct users, is
dealt with below. For now you can follow along with the Paperclip docs and do
a simple implementation. You can find a lot of detailed information about how
to wire up Paperclip to use S3 [here]
(http://www.rubydoc.info/gems/paperclip/Paperclip/Storage/S3)

Per the Paperclip docs, we need to make some changes to our models. Here is
how I have things set up.

```ruby
has_attached_file :file_attachment, s3_permissions: private

validates_attachment_content_type :file_attachment,
                                  content_type: ["image/jpeg", "image/png"]
```

##Angular
Now that we have this working on the Rails side, let's make it work with
Angular.

###ngFileUpload
For this article we will use the
[ngFileUpload directive](https://github.com/danialfarid/ng-file-upload) to
create the form and send the file to the server. The directive
is not specifically made to work with Paperclip or Rails, so we have to do some
extra work.

First, the markup.

```html
<div class="modal-body">
  <form role="form" name="resourceForm" ng-submit="save()">
    <div class="form-group">
      <label>Title</label>
      <input type="text" class="form-control" ng-required="true"
            ng-model="myResource.title" placeholder="" />
    </div>
    <!--some other fields here -->
    <div class="form-group">
      <label>Attachment</label>
      <input type="file" ngf-select ng-model="myResource.file_attachment" />
    </div>
  </form>
</div>
```

As you can see I am inserting the ngf-select directive in a form in a modal
window. When the user submits the form, the save() method in my controller
passes the form data to a service that takes the form data, and optionally
the id of the resource, in the case of editing the resource. I think these
should really be combined into a single method, but it works just fine like
this. Notice I do not trust the id coming from the form, but instead get it from
the resource attached to $scope. Of course on the Rails side we check user
authorization as well, so a user could not just change the id of a resource
and edit something they shouldn't. For this article I won't include my
authorization code.

```coffeescript
    MyResources.createWithAttachment(formData)
```

or

```coffeescript
    MyResources.editWithAttachment(formData, $scope.myResource.id)
```

This particular project uses Restangular, here is my upload service.

```coffeescript
angular.module("service")

  .factory "MyResources", ["Upload", (Upload) ->

    sendPayload = (formData, method, url) ->
      file_attachment = formData.file_attachment ? []
      options =
        url: url
        method: method
        file: file_attachment
        file_form_data_name: file_attachment.name ? ""
        fields:
          my_resource:
            title: formData.title
            body: formData.body

      Upload.upload(options)

    createWithAttachment: (formData) ->
      sendPayload(formData,
                  "POST",
                  "https://myapi.com/my_resources.json")

    editWithAttachment: (formData, recordId) ->
      sendPayload(formData,
                  "PUT",
                  "https://myapi.com/my_resources/#{recordId}.json")
  ]
```

##Rails Controllers
Because we are using the Upload object to make the API call, and not Restangular
like we are through the rest of the app, I needed to manipulate the request
parameters a bit. There are two steps to this.
First, parsing the nested resource hash into json,
then creating a Ruby hash that will be have like the normal params hash. This
is necessary because nesting my_resource under the fields key when setting
options for the JavaScript Upload object (lines 13-15 above) results in this
data not being properly serialized. So we do it manually.

Second, I nest the file parameter under the my_resource
parameter as it would naturally be using the Paperclip Rails view helpers.
These two steps make it easy to use the payload params just as I normally would.

I am omitting a lot of code for clarity -- code to check authorization, handle
save failures, etc.

```ruby
class MyResourcesController < ApplicationController
  before_filter :process_params, only: [:create, :update]

  def create
    my_resource = MyResource.new(permitted_params)
    my_resource.save
  end

  private
    def permitted_params
      params.require(:my_resource).permit(:title, :body, :file_attachment)
    end

    def process_params
      params[:my_resource] = JSON.parse(params[:my_resource])
                                 .with_indifferent_access
      if params[:file]
        params[:my_resource][:file_attachment] = params[:file]
      end
    end
end
```

##Secure downloads
By now we should be able to create and edit file attachments. Allowing
users to download them securely requires an additional step. We will do things
similarly to how [ThoughtBot recommends]
(https://github.com/thoughtbot/paperclip/wiki/Restricting-Access-to-Objects-Stored-on-Amazon-S3).
First we will define a method in our controller to generate a
secure download url for the resource. To do this we'll use a method provided by
Paperclip that creates a hard-to-guess expiring downloadable url for the resource.

```ruby
  def file_attachment_url
    my_resource = MyResource.find(params[:id])
    if my_resource
      render json: my_resource.file_attachment.expiring_url(10),
             status: 200,
             root: false
    end
  end
```

And we'll update our config/routes.rb file to expose this method.

```ruby
resources :my_resources do
  member do
    get "file_attachment_url"
  end
end
```

Now we'll create a simple directive in Angular that will let us use this
endpoint to fetch an expiring url and then redirect the user to it. Here we're
using the customGET() method provided by Restangular. When a user clicks this
element, we get a fresh url and redirect the browser.

```coffeescript
angular.module("shared")

  .directive "fileDownload", ["$window", ($window) ->
    restrict: "A"
    scope:
      record: "="
    link: ($scope, element) ->
      element.on "click", ->
        $scope.record.customGET("file_attachment_url").then (response) ->
          $window.open(response)
  ]
```

And, we use our new directive to display a link to the file.

```html
<a file-download record="myResource">
  {{myResource.file_attachment_file_name}}
</a>
```

And we're done!
