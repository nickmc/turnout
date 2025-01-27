Turnout [![Build Status](https://travis-ci.org/biola/turnout.svg?branch=master)](https://travis-ci.org/biola/turnout) [![Code Climate](https://codeclimate.com/github/biola/turnout.svg)](https://codeclimate.com/github/biola/turnout) [![Gem Version](https://badge.fury.io/rb/turnout.svg)](https://badge.fury.io/rb/turnout)
=======
Turnout is [Rack](http://rack.rubyforge.org/) middleware with a [Ruby on Rails](http://rubyonrails.org) engine that allows you to easily put your app in maintenance mode.

This fork
=======
This version is forked to allow whitelisting of user_ids. It adds config option of 'allowed_user_ids'. 
Any session with this user ud will be allowed to access the site.
To use this, you will want to whitelist the login page as well.

Features
========
* Easy installation
* Rake commands to turn maintenance mode on and off
* Easily provide a reason for each downtime without editing the maintenance.html file
* Allow certain IPs or IP ranges to bypass the maintenance page
* Allow certain paths to be accessible during maintenance
* Easily override the default maintenance.html file with your own
* Simple [YAML](http://yaml.org) based config file for easy activation, deactivation and configuration without the rake commands
* Support for multiple maintenance page formats. Current [HTML](http://en.wikipedia.org/wiki/HTML) and [JSON](http://en.wikipedia.org/wiki/JSON)
* Supports Rails, [Sinatra](http://sinatrarb.com) and any other Rack application
* Supports multiple maintenance file paths so that groups of applications can be put into maintenance mode at once.

Installation
============
Rails 3+
--------
In your `Gemfile` add:

    gem 'turnout'

then run

    bundle install
    
_Note that you'll need to restart your Rails server before it will work_

Sinatra
-------

In your Sinatra app file

```ruby
require 'rack/turnout'

class App < Sinatra::Base
  configure do
    use Rack::Turnout
```

In your Rakefile

```ruby
require 'turnout/rake_tasks'
```

Activation
==========

    rake maintenance:start

or

    rake maintenance:start reason="Somebody googled Google!"

or

    rake maintenance:start allowed_paths="/login,^/faqs/[0-9]*"

or

    rake maintenance:start allowed_ips="4.8.15.16"

or

    rake maintenance:start reason="Someone told me I should type <code>sudo rm -rf /</code>" allowed_paths="^/help,^/contact_us" allowed_ips="127.0.0.1,192.168.0.0/24"

or if you've configured `named_maintenance_file_paths` with a path named `server`

    rake maintenance:server:start

Notes
-----
* The `reason` parameter can contain HTML
* Multiple `allowed_paths` and `allowed_ips` can be given. Just comma separate them.
* All `allowed_paths` are treated as regular expressions.
* If you need to use a comma in an `allowed_paths` regular expression just escape it with a backslash: `\,`.
* IP ranges can be given to `allowed_ips` using [CIDR notation](http://en.wikipedia.org/wiki/CIDR_notation).

Deactivation
============

    rake maintenance:end

or if you activated with a named path like `server`

    rake maintenance:server:end

Configuration
=============

Turnout can be configured in two different ways:

1. __Pass a config hash to the middleware__

    ```ruby
    use Rack::Turnout,
      app_root: '/some/path',
      named_maintenance_file_paths: {app: 'tmp/app.yml', server: '/tmp/server.yml'},
      maintenance_pages_path: 'app/views/maintenance',
      default_maintenance_page: Turnout::MaintenancePage::JSON,
      default_reason: 'Somebody googled Google!',
      default_allowed_paths: ['^/admin/'],
      default_response_code: 418,
      default_retry_after: 3600
    ```

2. __Using a config block__

    ```ruby
    Turnout.configure do |config|
      config.skip_middleware = true
      config.app_root = '/some/path'
      config.named_maintenance_file_paths = {app: 'tmp/app.yml', server: '/tmp/server.yml'}
      config.maintenance_pages_path = 'app/views/maintenance'
      config.default_maintenance_page = Turnout::MaintenancePage::JSON
      config.default_reason = 'Somebody googled Google!'
      config.default_allowed_paths = ['^/admin/']
      config.default_response_code = 418
      config.default_retry_after = 3600
    end
    ```

__NOTICE:__ Any custom configuration should be loaded not only in the app but in the rake task. This should happen automatically in Rails as the `environment` task is run if it exists. But you may want to create your own `environment` task in non-Rails apps.

Default Configuration
---------------------

```ruby
Turnout.configure do |config|
  config.app_root = '.'
  config.named_maintenance_file_paths = {default: config.app_root.join('tmp', 'maintenance.yml').to_s}
  config.maintenance_pages_path = config.app_root.join('public').to_s
  config.default_maintenance_page = Turnout::MaintenancePage::HTML
  config.default_reason = "The site is temporarily down for maintenance.\nPlease check back soon."
  config.default_allowed_paths = []
  config.default_response_code = 503
  config.default_retry_after = 7200
end
```

Customization
=============

[Default maintenance pages](https://github.com/biola/turnout/blob/master/public/) are provided, but you can create your own `public/maintenance.[html|json|html.erb]` files instead. If you provide a `reason` to the rake task, Turnout will parse the maintenance page file and attempt to replace a [Liquid](http://liquidmarkup.org/)-style `{{ reason }}` tag with the provided reason. So be sure to include a `{{ reason }}` tag in your `maintenance.html` file. In the case of a `.html.erb` file, `reason` will be a local variable.

__WARNING:__
The source code of any custom maintenance files you created in the `/public` directory will be able to be viewed by visiting that URL directly (i.e. `http://example.com/maintenance.html.erb`). This shouldn't be an issue with HTML and JSON files but with ERB files, it could be. If you're going to use a custom `.erb.html` file, we recommend you change the `maintenance_pages_path` setting to something other than the `/public` directory.

Tips
====

Denied Paths
--------------
There is no `denied_paths` feature because turnout denies everything by default.
However you can achieve the same sort of functionality by using
[negative lookaheads](http://www.regular-expressions.info/lookaround.html) with the `allowed_paths` setting, like so:

    rake maintenance:start allowed_paths="^(?!/your/under/maintenance/path)"

Multi-App Maintenance
------------------------
A central `named_maintenance_file_path` can be configured in all your apps such as `/tmp/turnout.yml` so that all apps on a server can be put into mainteance mode at once. You could even configure service based paths such as `/tmp/mongodb_maintenance.yml` so that all apps using MongoDB could be put into maintenance mode.

Detecting Maintenance Mode
-------------------------------

If you'd like to detect if maintenance mode is on in your app (for those users or pages that aren't blocked) just call `!Turnout::MaintenanceFile.find.nil?`.

Behind the Scenes
=================
On every request the Rack app will check to see if `tmp/maintenance.yml` exists. If the file exists the maintenance page will be shown (unless allowed IPs are given and the requester is in the allowed range).

So if you want to get the maintenance page up or down in a hurry `touch tmp/maintenance.yml` and `rm tmp/maintenance.yml` will work.

Turnout will attempt to parse the `maintenance.yml` file looking for `reason`, `allowed_ip` and other settings. The file is checked on every request so you can change these values manually or just rerun the `rake maintenance:start` command.

Example maintenance.yml File
----------------------------

```yaml
---
reason: Someone told me I should type <code>sudo rm -rf /</code>
allowed_paths:
- ^/help
- ^/contact_us
allowed_ips:
- 127.0.0.1
- 192.168.0.0/24
response_code: 503
retry_after: 3600
```
