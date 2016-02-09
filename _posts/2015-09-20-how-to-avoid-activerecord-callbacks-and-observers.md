---
layout: post
title: How to avoid using ActiveRecord callbacks and Observers
---

A plethora of literature exists in the Rails world to dissuade you from using ActiveRecord callbacks. Callbacks are not intrinsically bad, but they give you a lot of rope to hang yourself with, and consequently they are an often abused feature of Rails.

Callbacks are attractive because they make certain tasks look and feel deceptively easy. But this is usually because the developer has not thoroughly thought through the consequences or the use cases, or because the application is still in its infancy when many of the pieces are yet to fallen into place.

## Callbacks obfuscate the intention of your code

Good code is explicit code. Using the example below, if you call `@user.save` you should mean to save the user instance only, and not save it and send an email afterwards. The code should express its intent clearly. Sending out the email should not be a side effect of saving the user.

{% highlight ruby %}
class User < ActiveRecord::Base
  after_create :send_welcome_email

private

  def send_welcome_email
    UserMailer.welcome_email(self).deliver_later
  end
end
{% endhighlight %}

To fix this, let's backtrack through the application, from the model back to the controller.

{% highlight ruby %}
class UsersController < ApplicatinController::Base
  def create
    @user = User.new(user_params)
    if @user.save
      UserMailer.welcome_email(@user).deliver_later
      redirect_to @user
    else
      render :new
    end
  end
end
{% endhighlight %}

This is an improvement, but it's not ideal for large applications. We have to remember to use the mailer in other places where it makes sense to send an email after creating a user (for example, from an admin panel or through an API call). In a way, this is the opposite of using a callback; in that, if using a callback means we send an email every time we create a user, moving the logic to the controller means we only send an email in one place. We need a more flexible way, a middle ground.

### Service objects as an alternative

A service object gives us this flexibility we're after. I prefer a service object because it makes our code explicit and reusable.

{% highlight ruby %}
# app/services/create_user.rb
class CreateUser
  attr_reader :user, :mailer

  def initialize(user, mailer: UserMailer)
    @user = user
    @mailer = mailer
  end

  def call
    if user.save
      mailer.welcome_email(user).deliver_later
    end
  end
end
{% endhighlight %}

Our service object can be used anywhere we need to create users and send out email notifications. And now we can create users free from side effects in other parts of the application whether it's from the console, a rake task, or somewhere else.

{% highlight ruby %}
class UsersController < ApplicatinController::Base
  def create
    @user = User.new(user_params)
    service = CreateUser.new(@user)
    if service.call
      redirect_to @user
    else
      render :new
    end
  end
end
{% endhighlight %}

If our requirements change to include an activity feed, extending the functionality of the service object is easy, and we only have to make a change in one place. Imagine the pain of using a callback to add the activity.

{% highlight ruby %}
class CreateUser
  # ...
  def call
    if user.save
      mailer.welcome_email(user).deliver_later
      UserActivityJob.perform_later(user)
    end
  end
end
{% endhighlight %}

### Alternatives to service objects

Service objects help us adhere to the [Single Responsibility Principle][1], and by extension improve our code organisation. But if service objects are not your thing, you can still avoid using a callback by using a method in your model.

{% highlight ruby %}
class User < ActiveRecord::Base
  def save_and_deliver_email(mailer: UserMailer)
    if save
      mailer.welcome_email(self).deliver_later
    end
  end
end
{% endhighlight %}

Although this is an improvement on callbacks, I don't like this approach because the model should not concern itself or know about mailers. The model is responsible for persistence and its internal business logic.

Concerns also give us a way to implement this feature without using callbacks, but having outlined service objects and their benefits, I don't see an advantage to using a concern.

{% highlight ruby %}
module UserRegistration
  extend ActiveSupport::Concern

  def save_user_and_deliver_email(user, mailer: UserMailer)
    if user.save
      mailer.welcome_email(user).deliver_later
    end
  end
end
{% endhighlight %}

Technically, we don't have to pass the user as an argument; concerns have access to instance variables set in controllers they're mixed into, but I consider this an anti-pattern. Passing an argument is a cheap price to pay for code clarity.

{% highlight ruby %}
class UsersController < ApplicatinController::Base
  include UserRegistration
  def create
    @user = User.new(user_params)
    if save_user_and_deliver_email(@user)
      redirect_to @user
    else
      render :new
    end
  end
end
{% endhighlight %}

## Callbacks make testing harder

Callbacks make your tests slower to execute and write, and can lead to failures that are hard to debug. Given our example, imagine large applications where you have hundreds of tests that require a user object. Each time you create a user you either incur the cost of the callback, or stub the method.

It is true that when testing the controller, or unit testing the service object, we may still have to stub the mailer. But we only have to do this in one place as opposed to hundreds.

## What about observers

Observers use the same pattern as callbacks, but the implementation details are slightly different. You can think of them as callbacks declared outside of the model. And because of this they suffer from the same problems that callbacks do.

## When do I use callbacks?

I only use callbacks when dealing with the internal state of the object. For example:

{% highlight ruby %}
class Invitation < ActiveRecord::Base
  before_create :set_token, :downcase_email

private

  def downcase_email
    email.downcase!
  end

  def set_token
    # A database uniqueness constraint will prevent a clash
    self.token = SecureRandom.urlsafe_base64
  end
 end
{% endhighlight %}

But even then I look for ways to avoid them. For example, but it's better to use [a setter method][2] to downcase the email.

{% highlight ruby %}
class User < ActiveRecord::Base
  before_create :set_token

  def email=(value)
    super(value.downcase)
  end

  # ...
end
{% endhighlight %}

Callbacks have their uses, but it's good to ask yourself if there's a better way to achieve what you are after.


[1]: https://en.wikipedia.org/wiki/Single_responsibility_principle
[2]: https://github.com/rails/rails/pull/19787/files
