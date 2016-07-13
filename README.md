## Introduction
Capistrano is a tool for automating the execution of commands on remote servers through SSH. Nginx is a light, high performance web server software. Passenger is the application server and PostgreSQL is the database server. ROR application and capistrano can be easily configured to work very well together on a virtual private server when installed through Phusion Passenger.

## Step 1 — Configure Rails Application

### 1 Initializing Capistrano
We need to configure our application for deployment. As previously mentioned, Passenger is the application server and Capistrano as our deployment tool. Capistrano provides integration for Passenger and RVM, so we add those gems to the Gemfile.

```
group :development do
  gem 'capistrano'
  gem 'capistrano-bundler'
  gem 'capistrano-passenger'
  gem 'capistrano-rails'
  gem 'capistrano-rvm'
end
```

Install the gems:
```
bundle install
````

Generate the config file:
```
cap install STAGES=production
```
This will create some configuration files like config/deploy.rb , config/deploy/production.rb and Capfile. deploy.rb is the main configuration file, production.rb contains environment specific settings, such as server IP, username, etc and Capfile is the Capistrano entry point. It defines what recipes to load. You must edit it to load the recipes you need.

### 2 Editing Capfile
If you open Capfile you will see:
```
# Load DSL and set up stages
require 'capistrano/setup'

# Include default deployment tasks
require 'capistrano/deploy'

# Include tasks from other gems included in your Gemfile
#
# For documentation on these, see for example:
#
#   https://github.com/capistrano/rvm
#   https://github.com/capistrano/rbenv
#   https://github.com/capistrano/chruby
#   https://github.com/capistrano/bundler
#   https://github.com/capistrano/rails
#   https://github.com/capistrano/passenger
#
# require 'capistrano/rvm'
# require 'capistrano/rbenv'
# require 'capistrano/chruby'
# require 'capistrano/bundler'
# require 'capistrano/rails/assets'
# require 'capistrano/rails/migrations'
# require 'capistrano/passenger'

# Load custom tasks from `lib/capistrano/tasks' if you have any defined
Dir.glob('lib/capistrano/tasks/*.rake').each { |r| import r }
```

We want Capistrano to automatically run bundle install, and we want Capistrano to automatically tell Passenger to restart our app. We need to make sure that we uncomment the following lines:
```
require 'capistrano/rvm'
require 'capistrano/bundler'
require 'capistrano/passenger'
require 'capistrano/rails/assets'
require 'capistrano/rails/migrations'
```

### 3 Editing config/deploy.rb
The next step is to edit config/deploy.rb. This file contains configuration values that control how the loaded recipes should do their jobs. It also defines additional commands to be executed on servers.

If you open the file, we will see this:
```
# config valid only for current version of Capistrano
lock '3.3.5'

set :application, 'my_app_name'
set :repo_url, 'git@example.com:me/my_repo.git'

# Default branch is :master
# ask :branch, proc { `git rev-parse --abbrev-ref HEAD`.chomp }.call

# Default deploy_to directory is /var/www/my_app_name
# set :deploy_to, '/var/www/my_app_name'

# Default value for :scm is :git
# set :scm, :git

# Default value for :format is :pretty
# set :format, :pretty

# Default value for :log_level is :debug
# set :log_level, :debug

# Default value for :pty is false
# set :pty, true

# Default value for :linked_files is []
# set :linked_files, fetch(:linked_files, []).push('config/database.yml')

# Default value for linked_dirs is []
# set :linked_dirs, fetch(:linked_dirs, []).push('bin', 'log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', 'public/system')

# Default value for default_env is {}
# set :default_env, { path: "/opt/ruby/bin:$PATH" }

# Default value for keep_releases is 5
# set :keep_releases, 5

namespace :deploy do

  after :restart, :clear_cache do
    on roles(:web), in: :groups, limit: 3, wait: 10 do
      # Here we can do anything such as:
      # within release_path do
      #   execute :rake, 'cache:clear'
      # end
    end
  end

end
```

There are a few things we see here.

##### Variables
* deploy_to: It tells us where to deploy the application. In our case we set it to ~/www/hello
* application: used by the capistrano/deploy recipe to determine which directory to deploy to. So if we set application to hello, Capistrano will deploy to ~/www/hello
* repo_url: used by the capistrano/deploy recipe to determine where to clone/pull code from.
* linked_files and linked_dirs: used by the capistrano/deploy recipe to link files and directories from the shared directory into a release directory.

##### Tasks
Inside the namespace block, we can define additional commands to execute during a deployment process. Typical Capistrano script that has a few popular recipes loaded, perform the following steps:

* Cloning code into a release directory
* Setting up linked files and directory
* Running bundle install, running database migrations, etc
* Changing the current symlink
* Restarting the application server (Passenger, in our case)

Now we uncomment and edit the following lines in config/deploy.rb file
```
set :linked_files, fetch(:linked_files, []).push('config/database.yml', 'config/secrets.yml')
set :linked_dirs, fetch(:linked_dirs, []).push('log', 'tmp/pids', 'tmp/cache', 'tmp/sockets', 'vendor/bundle', 'public/system', 'public/uploads')
```

Set rvm_ruby_version to the Ruby version that our application uses. For example:
```
set :rvm_ruby_version, '2.2.1'
```

### 4 Editing config/deploy/production.rb
This file defines the servers that Capistrano should deploy to, in the form of SSH login information.

If we open the file, we see this:

```
# Simple Role Syntax
# ==================
# Supports bulk-adding hosts to roles, the primary server in each group
# is considered to be the first unless any hosts have the primary
# property set.  Don't declare `role :all`, it's a meta role.

role :app, %w{deploy@example.com}
role :web, %w{deploy@example.com}
role :db,  %w{deploy@example.com}


# Extended Server Syntax
# ======================
# This can be used to drop a more detailed server definition into the
# server list. The second argument is a, or duck-types, Hash and is
# used to set extended properties on the server.

server 'example.com', user: 'deploy', roles: %w{web app}, my_property: :my_value


# Custom SSH Options
# ==================
# You may pass any option but keep in mind that net/ssh understands a
# limited set of options, consult[net/ssh documentation](http://net-ssh.github.io/net-ssh/classes/Net/SSH.html#method-c-start).
#
# Global options
# --------------
#  set :ssh_options, {
#    keys: %w(/home/rlisowski/.ssh/id_rsa),
#    forward_agent: false,
#    auth_methods: %w(password)
#  }
#
# And/or per server (overrides global)
# ------------------------------------
# server 'example.com',
#   user: 'user_name',
#   roles: %w{web app},
#   ssh_options: {
#     user: 'user_name', # overrides user setting above
#     keys: %w(/home/user_name/.ssh/id_rsa),
#     forward_agent: false,
#     auth_methods: %w(publickey password)
#     # password: 'please use keys'
#   }
```

Change web, app and db with yor server address.

### 5 Preparing the server
