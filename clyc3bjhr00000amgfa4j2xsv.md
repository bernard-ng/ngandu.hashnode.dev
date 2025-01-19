---
title: "Deploying Ruby on Rails with Capistrano, Puma, and Nginx Simplified"
seoTitle: "Deploy Rails with Capistrano, Puma, Nginx"
seoDescription: "Deploy Ruby on Rails with Capistrano, Puma, and Nginx on an Ubuntu VPS, simplifying the deployment process step-by-step"
datePublished: Sun Jul 07 2024 21:53:36 GMT+0000 (Coordinated Universal Time)
cuid: clyc3bjhr00000amgfa4j2xsv
slug: deploying-ruby-on-rails-with-capistrano-puma-and-nginx-simplified
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1720377616998/556cc0bd-c7ea-491d-91d9-6faa0ef035a1.jpeg
tags: capistrano, ruby, nginx, ruby-on-rails

---

I recently completed the development of my first [Ruby on Rails](https://rubyonrails.org/) 7.1 application. As you know, once development is complete, comes the crucial deployment stage. In my case, I chose to deploy on an [Ubuntu](https://ubuntu.com/) VPS hosted on [AWS LightSail](https://aws.amazon.com/lightsail/). After hours of research and debugging, I've decided to share here a summary of everything I've learned in the process.

## Context

Basically, I've developed an application that communicates with an API, whose URLs are defined in a `.env` file. For the database, I opted for [SQLite](https://www.sqlite.org/). Yes, you can use SQLite in production! Just don't forget to add `config.active_record.sqlite3_production_warning = false` in **config/environments/production.rb** to get rid of warnings !

There are many ways and tools available for deployment. In this article, we'll use Capistrano, Nginx, and Puma.

* [**Capistrano**](https://capistranorb.com/): A remote server automation and deployment tool written in Ruby.
    
* [Puma](https://puma.io/): A fast, concurrent web server for ruby & rack.
    
* [Nginx](https://nginx.org/en/): a web server that acts as a reverse proxy, directing requests to the Ruby on Rails application.
    

Capistrano will allow us to automate the entire deployment process. Note that [SSH](https://en.wikipedia.org/wiki/Secure_Shell) access to the server on which you wish to deploy is essential, and that the code to be deployed must be versioned, for example with [Git](https://git-scm.com/).

## Let's do it !

### Prepare the remote server

First of all, make sure you have the same version of Ruby installed on your server as on your development environment. To manage this easily, you can use [RVM](https://rvm.io/) or [rbenv](https://rbenv.org/). Refer to the documentation for the tool you have chosen. Once Ruby has been installed, you'll also need to install [Bundler](https://bundler.io/), which will enable you to install the application's various dependencies.

```bash
gem install bundler
```

I use [yarn](https://yarnpkg.com/) and [webpack](https://webpack.js.org/) to build assets. If this is also your case, be sure to install [Node.js](https://nodejs.org/en) as well as any other tools essential to the smooth running of your application (redis, mysql, etc...).

### Installing Gems

Now that everything's in place, we need to install the following gems. Note that version constraints are very important to avoid compatibility problems. Believe me, you don't want to have to deal with these worries.

```ruby
# Gemfile
gem 'puma', '~> 6.0.0', '< 7'
gem 'dotenv-rails' # to handle .env files
gem 'sd_notify', '~> 0.1.0' # hotfix not actually used

group :development do
  gem 'capistrano'
  gem 'capistrano3-puma', '6.0.0.beta.1' # supports puma 6+
  gem 'capistrano-rails'
  gem 'capistrano-rvm' # capistrano-rbenv for rbenv
end
```

Then run `bundle install` to install the gems with the specified versions.

### Setting up Capistrano

```bash
cap install
```

This command will create several files and folders in your project:

* **Capfile**
    
* **config/deploy.rb**
    
* **config/deploy/production.rb**
    
* **config/deploy/staging.rb**
    

Let's modify the **Capfile** to include the necessary plugins

```ruby
# frozen_string_literal: true

require 'capistrano/setup'
require 'capistrano/deploy'
require 'capistrano/scm/git'
install_plugin Capistrano::SCM::Git

# Include tasks from other gems included in your Gemfile
require 'capistrano/rvm' # 'capistrano/rbenv' for rbenv
require 'capistrano/bundler'
require 'capistrano/rails' # will include assets and migrations tasks
require 'capistrano/puma'
require 'capistrano/puma/nginx'
install_plugin Capistrano::Puma
install_plugin Capistrano::Puma::Systemd

# Load custom tasks from `lib/capistrano/tasks` if you have any defined
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

To customize the deployment, we need to modify the **config/deploy.rb** file.

```ruby
# frozen_string_literal: true
lock '~> 3.19.1'

set :application, 'ecovet.cloud' # your app name
set :repo_url, 'git@github.com:bernard-ng/ecovet.cloud.git'
set :branch, 'main'
set :deploy_to, '/var/www/html/ecovet.cloud' # path on remote server

set :pty, true
append :linked_files, 'config/master.key', '.env', 'config/database.yaml'
append :linked_dirs, 'log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor', 'storage'

set :keep_releases, 2
set :ssh_options, {
  forward_agent: true,
  auth_methods: %w[publickey],
  keys: %w[~/.ssh/LightsailDefaultKey-eu-west-2.pem]
}

# puma
set :puma_workers, 2 # check your CPU specs
set :puma_rackup, -> { File.join(current_path, 'config.ru') }
set :puma_state, "#{shared_path}/tmp/pids/puma.state"
set :puma_pid, "#{shared_path}/tmp/pids/puma.pid"
set :puma_bind, "unix://#{shared_path}/tmp/sockets/puma.sock"
set :puma_default_control_app, "unix://#{shared_path}/tmp/sockets/pumactl.sock"
set :puma_access_log, "#{shared_path}/log/puma_access.log"
set :puma_error_log, "#{shared_path}/log/puma_error.log"
set :puma_conf, "#{shared_path}/puma.rb"

set :puma_control_app, false
set :puma_systemctl_user, :system
set :puma_service_unit_type, 'simple' # or notify
set :puma_enable_socket_service, true # mendatory in our case

# nginx
set :nginx_config_name, 'ecovet.cloud'
set :nginx_server_name, 'ecovet.cloud'
set :nginx_use_ssl, false # will be handled by certbot
```

Configure your production environment in **config/deploy/production.rb** 👇🏾

```ruby
# frozen_string_literal: true
# config/deploy/production.rb

server 'ecovet.cloud', user: 'ubuntu', roles: %w[app db web], ssh_options: { forward_agent: true }
```

### Configuration Templates

As you may have guessed, it's crucial that Puma, our web server, is always up and running. Even if the machine reboots or crashes, Puma must restart itself automatically. To ensure this, we can use [systemd](https://en.wikipedia.org/wiki/Systemd), a service management system that starts, stops and manages processes automatically.

Configuring Puma with systemd is divided into two distinct parts:

1. The first concerns the definition of the Puma service, where we specify how it should be started, which user should run it, and how it should be restarted if necessary.
    
2. The second part concerns the configuration of the Puma socket, which handles communication between Puma and Nginx or any other component using the socket to transmit HTTP requests.
    

Here are the configuration templates you should add to your project, adapting them to your needs if necessary. Currently, the capistrano3-puma gem has not been updated for two years, which may cause problems with its default configuration.

* **config/deploy/templates/puma.rb.erb**
    
* **config/deploy/templates/puma.service.erb**
    
* **config/deploy/templates/puma.socket.erb,** you must `set :puma_enable_socket_service, true` in **config/deploy.rb** otherwise you'll have to create the socket service manually.
    

**config/deploy/templates/puma.rb.erb** 👇🏾

```erb
#!/usr/bin/env puma

directory '<%= current_path %>'
rackup "<%=fetch(:puma_rackup)%>"
environment '<%= fetch(:puma_env) %>'
<% if fetch(:puma_tag) %>
  tag '<%= fetch(:puma_tag)%>'
<% end %>
pidfile "<%=fetch(:puma_pid)%>"
state_path "<%=fetch(:puma_state)%>"
stdout_redirect '<%=fetch(:puma_access_log)%>', '<%=fetch(:puma_error_log)%>', true


threads <%=fetch(:puma_threads).join(',')%>

<%= puma_bind %>
<% if fetch(:puma_control_app) %>
activate_control_app "<%= fetch(:puma_default_control_app) %>"
<% end %>
workers <%= puma_workers %>
<% if fetch(:puma_worker_timeout) %>
worker_timeout <%= fetch(:puma_worker_timeout).to_i %>
<% end %>

<% if puma_preload_app? %>
preload_app!
<% else %>
prune_bundler
<% end %>

on_restart do
  puts 'Refreshing Gemfile'
  ENV["BUNDLE_GEMFILE"] = "<%= fetch(:bundle_gemfile, "#{current_path}/Gemfile") %>"
end

<% if puma_preload_app? and fetch(:puma_init_active_record) %>
on_worker_boot do
  ActiveSupport.on_load(:active_record) do
    ActiveRecord::Base.establish_connection
  end
end
<% end %>
```

**config/deploy/templates/puma.service.erb** 👇🏾

```erb
[Unit]
Description=Puma HTTP Server for <%= "#{fetch(:application)} (#{fetch(:stage)})" %>
<%= "Requires=#{fetch(:puma_service_unit_name)}.socket" if fetch(:puma_enable_socket_service) %>
After=syslog.target network.target

[Service]
Type=<%= service_unit_type %>
WatchdogSec=10
<%="User=#{puma_user(@role)}" if fetch(:puma_systemctl_user) == :system %>
WorkingDirectory=<%= current_path %>
ExecStart=<%= expanded_bundle_command %> exec --keep-file-descriptors puma -e <%= fetch(:puma_env) %> -C /var/www/html/ecovet.cloud/shared/puma.rb
ExecReload=/bin/kill -USR1 $MAINPID
PIDFile=<%= fetch(:puma_pid)%>
<%- Array(fetch(:puma_service_unit_env_files)).each do |file| %>
<%="EnvironmentFile=#{file}" -%>
<% end -%>
<% Array(fetch(:puma_service_unit_env_vars)).each do |environment_variable| %>
<%="Environment=\"#{environment_variable}\"" -%>
<% end -%>

# if we crash, restart
RestartSec=1
Restart=on-failure

<%="StandardOutput=append:#{fetch(:puma_access_log)}" if fetch(:puma_access_log) %>
<%="StandardError=append:#{fetch(:puma_error_log)}" if fetch(:puma_error_log) %>

SyslogIdentifier=<%= fetch(:puma_service_unit_name) %>
[Install]
WantedBy=<%=(fetch(:puma_systemctl_user) == :system) ? "multi-user.target" : "default.target"%>
```

**config/deploy/templates/puma.socket.erb** 👇🏾

```erb
[Unit]
Description=Puma HTTP Server Accept Sockets for <%= "#{fetch(:application)} (#{fetch(:stage)})" %>

[Socket]
<% puma_binds.each do |bind| -%>
<%= "ListenStream=#{bind.local.address}" %>
<% end -%>

Accept=no
<%= "NoDelay=true" if fetch(:puma_systemctl_user) == :system %>
ReusePort=true
Backlog=1024

SyslogIdentifier=puma_socket

[Install]
WantedBy=sockets.target
```

### Uploading Nginx and Puma configuration

Once you've added these configurations, you can send them to your remote server using the following commands with Capistrano. Note that if your Puma, Nginx or systemd configuration hasn't changed, this step won't be necessary.

```bash
cap production puma:config # will upload puma.rb
cap production puma:nginx_config # will upload nginx config
cap production puma:install # will create systemd service and stocket
cap production puma:start # will start the puma service
```

### .env support

Edit your **config/application.rb** to add support for .env files

```ruby
# ...
# Load .env file
Dotenv::Rails.load
```

### HTTPS support

Let’s Encrypt is a Certificate Authority (CA) that provides an easy way to obtain and install free [TLS/SSL certificates, thereby enabling e](https://www.digitalocean.com/community/tutorials/openssl-essentials-working-with-ssl-certificates-private-keys-and-csrs)ncrypted HTTPS on web servers. It simplifies the process by providing a software client, [Certbot](https://certbot.eff.org/), that attempts to automate most (if not all) of the required steps. Currently, the entire process of obtaining and installing a certificate is fully automated on both Apache and Nginx.

```bash
# remote server
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx
sudo service nginx reload
```

## Deploy !!!

Now that everything is ready, we can move on to deployment with the command

```bash
cap production deploy
```

This command will perform several actions to ensure the complete and functional deployment of your application

1. The Git repository will be cloned into a new folder under `releases/{timestamp}` on your remote server.
    
2. **Symbolic links to the shared folder:** Files or directories defined in `linked_files` and `linked_dirs` in your Capistrano configuration will be symbolically linked from the `shared` folder to the `current` folder.
    
3. **Dependency installation:** Ruby dependencies defined in your `Gemfile` will be installed via `bundle install`.
    
4. **Pre-compiling assets:** Your application's assets will be pre-compiled with Yarn and Webpack if necessary, preparing your application for production deployment.
    
5. **Execute migrations:** Necessary database migrations will be launched via `bundle exec rails db:migrate` to update the database structure according to the changes made.
    
6. **Starting or restarting Puma:** Puma will be started or restarted using the specified configuration file (`config/puma.rb`), ensuring that your web application is up and running on the server.
    

# Conclusion

I hope this article saves you some time and sheds some light on the deployment process. [Ecovet.cloud](http://Ecovet.cloud) is now online, but being a study project, it won't be around for long! Happy coding!