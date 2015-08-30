---
layout: post
title: Adding a dynamic maintenance mode to a Rails app
---

During the past month we've been busy introducing new features and making changes to our application, Nimbus Work Spaces. On a few occasions we needed to temporarily disable access to the application while we carried out migrations and tests. The first couple of times we disabled access with a controller `before_action` that redirected to a maintenance page unless the current user id was whitelisted. But it soon became clear that having a proper and scalable maintenance strategy is necessary.

There are several ways to implement a maintenance mode, but they typically work by bypassing requests to the application server and serving a static HTML page instead. Heroku has a built-in [maintenace feature][1] that works in a similar way. When enabled it serves a static HTML template. Unfortuntely such solutions aren't dynamic, and thus fail our requirements. We want to:

1. Allow access to some users while the application is in maintenance mode.
2. Serve a custom, internationalized template.

This means the application has to be involved so it can determine how to handle the request, and what language to serve. We implemented this feature by adding a normal controller action that rendered the maintenance template.

{% highlight ruby %}
class DowntimeController < ApplicationController
  skip_before_action :check_maintenance_mode

  def show
  end
end
{% endhighlight %}

The only code of interest here is the `skip_before_action` call; it ensures that we don't check for maintenance mode when we are viewing the maintenance page. This stops the application from going into an infinite loop.

We also added a concern to handle the logic, and mixed it into `application_controller.rb`.

{% highlight ruby %}
module MaintenanceMode
  extend ActiveSupport::Concern

  included do
    before_action :check_maintenance_mode
  end

private

  def check_maintenance_mode
    if maintenance_mode?
      redirect_to maintenance_path unless permitted_ip?
    end
  end

  def maintenance_mode?
    ENV['MAINTENANCE_MODE']
  end

  def permitted_ip?
    maintainer_ips.split(',').include?(request.remote_ip)
  end

  def maintainer_ips
    ENV['MAINTAINER_IPS'] || String.new
  end
end
{% endhighlight %}

{% highlight ruby %}
class ApplicationController < ActionController::Base
  before_filter :check_maintenance
end
{% endhighlight %}

This module adds a `before_action` that checks if the application is in maintenance mode, and does any work required.

The feature is now complete. To use it, and enter maintenance mode, we set an `ENV` variable on Heroku.

{% highlight text %}
heroku config:set MAINTENANCE_MODE=enabled
{% endhighlight %}

It doesn't really matter what the value of `MAINTENANCE_MODE` is (or its name, for that matter), so *enabled* serves for clarity. The logic checks for the presence of the variable, not its value.

If we want to allow access to a specific IP address we set another `ENV` variable. Its value should be a coma-delimited list of any IP address that we want to enable access for.

{% highlight text %}
heroku config:set MAINTAINER_IPS=1.2.3.4,9.8.7.6
{% endhighlight %}

And finally, to exit maintenance mode.

{% highlight text %}
heroku config:unset MAINTENANCE_MODE
{% endhighlight %}

Our module is also very easy to test.

{% highlight ruby %}
require 'test_helper'

class MaintenanceModeTest < ActionDispatch::IntegrationTest
  setup do
    ENV['MAINTENANCE_MODE'] = 'enabled'
    ENV['MAINTAINER_IPS'] = '1.2.3.4'
  end

  teardown do
    ENV.delete('MAINTENANCE_MODE')
    ENV.delete('MAINTAINER_IPS')
  end

  test "redirects to maintenance path when in maintenance mode" do
    get root_path
    assert_redirected_to maintenance_path
  end

  test "does not redirect to maintenance path when IP is whitelisted" do
    get root_path, {}, { 'REMOTE_ADDR' => '1.2.3.4' }
    assert_equal root_path, path
  end
end
{% endhighlight %}

And there it is, a flexible and dynamic maintenance mode for your Rails application.

[1]: https://devcenter.heroku.com/articles/maintenance-mode
