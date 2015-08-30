---
layout: post
title: Adding a dynamic maintenance mode to a Rails app
---

It's a good idea to have a maintenance strategy for your Rails application. You may want to temporarily disable access to the application while you carry out upgrades and run other tasks like database or server migrations.

While there are several ways to implement this, they typically work by bypassing requests to the application server and serving a static HTML page instead. Heroku has a built-in [maintenace feature][1] that works in a similar way. When enabled it serves a static HTML template. Unfortuntely, such solutions fail our requirements. We want to:

1. Allow access to some users while the application is in maintenance mode.
2. Serve a custom, internationalized template.

This means the application has to be involved; so we migrated the process into it. First, we added a route.

	get '/maintenance', to: 'downtime#show'

We then added a corresponding controller and view to display to the user when the application is in maintenance mode. Now we can serve the page dynamically, and use I18n.

	class DowntimeController < ApplicationController
	  skip_before_action :check_maintenance_mode

	  def show
	  end
	end

The `skip_before_action` ensures that we don't check for maintenance mode when we are viewing the maintenance page. This stops the application from going into an infinite loop.

Finally we added a controller concern and mixed it into `application_controller.rb`.

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

	class ApplicationController < ActionController::Base
	  before_filter :check_maintenance
	end

This adds a `before_action` that checks if the application is in maintenance mode, and does any work required.

To enter maintenance mode we set an `ENV` variable on Heroku.

	heroku config:set MAINTENANCE_MODE=enabled

It doesn't really matter what the value of `MAINTENANCE_MODE` is, so *enabled* serves for clarity. The concern we added checks for the presence of the var and not its value.

To allow access to a specific IP we set another `ENV` variable, and set its value to a coma-delimited list of IPs that we want to allow access to.

	heroku config:set MAINTAINER_IPS=1.2.3.4,9.8.7.6

And finally, to exit maintenance mode.

	heroku config:unset MAINTENANCE_MODE

It's also very easy to test.

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

	  test "does not redirect to maintenance path when IP is white listed" do
	    get root_path, {}, { 'REMOTE_ADDR' => '1.2.3.4' }
	    assert_equal root_path, path
	  end
	end

[1]: https://devcenter.heroku.com/articles/maintenance-mode
