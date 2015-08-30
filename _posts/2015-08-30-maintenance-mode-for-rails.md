It's a good idea to have a maintenance strategy for your Rails application. You may want to temporarily disable access to the application while you carry out upgrades and run other tasks like database or server migrations.

There are several ways to implement this, and they all have their own caveats. For our requirements we wanted to:

1. Allow access to some users while the application is in maintenance mode.
2. Serve a custom, internationalized template.

Heroku has a [built-in maintenace feature][1] that serves a static maintenace HTML page. When it's enabled it reroutes all requests to a static HTML page, bypassing the application server entirely. This means the feature fails both of our requirements. You can create a custom page, but you can not internationalize it. This means we can't serve a Portuguese page for our Brazilian users, and an English one for our English speaking users.

We solved this by migrating the process into the application. First, we added a route.


```
get '/maintenance', to: 'downtime#show'
```

We then added a corresponding controller and view to display to the user when the application is in maintenance mode. This lets us serve the page dynamically, and use I18n.

```
class DowntimeController < ApplicationController
  skip_before_action :check_maintenance_mode

  def show
  end
end
```

The `skip_before_action` ensures that we don't check for maintenance mode when we are viewing the maintenance page. This stops the application from going into an infinite loop.

Then, we added a controller concern and mixed it into application controller.

```
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
```

This adds a `before_action` that checks if the application is in maintenance mode, and does any work required.

To enter maintenance mode we set an `ENV` variable on Heroku.

```
heroku config:set MAINTENANCE_MODE=enabled
```

It doesn't really matter what the value of `MAINTENANCE_MODE` is, so *enabled* serves for clarity. The concern we added checks for the presence of the var and not its value.

To allow access to a specific IP we set another `ENV` variable and set its value to a coma-delimited list of IPs that we want to allow access to.

```
heroku config:set MAINTAINER_IPS=1.2.3.4,9.8.7.6
```

And finally, to exit maintenance mode.


```
heroku config:unset MAINTENANCE_MODE
```

It's also very easy to test.

```ruby
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
```

[1]: https://devcenter.heroku.com/articles/maintenance-mode