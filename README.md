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


## Step 2 — Preparing the server

#### Create instance
Create an instance of the server and setup required users.

We need to setup a basic directory structure on the server, create initial configuration files and configure Passenger.

Please login to your server as an administrator, then follow the following instructions.

Update the existing packages first:

```
sudo apt-get update && sudo apt-get -y upgrade
```

#### Install Git

Git is required for automated deployments via Capistrano, so install Git on the server:
```
sudo apt-get install git
```

#### Setting up a basic directory structure
Run the following commands on the server to setup a basic directory structure that Capistrano can work with.

```
sudo mkdir -p /www/myapp/shared
```

#### Install Ruby with RVM
```
curl -L get.rvm.io | bash -s stable
source ~/.rvm/scripts/rvm
rvmsudo /usr/bin/apt-get install build-essential openssl libreadline6 libreadline6-dev curl git-core zlib1g zlib1g-dev libssl-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt-dev autoconf libc6-dev ncurses-dev automake libtool bison subversion

rvm install 2.2.1
rvm use 2.2.1 --default

rvm rubygems current
```
#### Install Rails
Install bundler and rails
```
gem install bundler --no-ri --no-rdoc
gem install rails
```

#### Install Passenger
```
gem install passenger 
```

#### Install nginx
```
rvmsudo passenger-install-nginx-module
```
And now Passenger takes over.

Passenger first checks that all of the dependancies it needs to work are installed. If you are missing any, Passenger will let you know how to install them, either with the apt-get installer on Ubuntu.

After you download any missing dependancies, restart the installation. Type: passenger-install-nginx-module once more into the command line.

Passenger offers users the choice between an automated setup or a customized one. Press 1 and enter to choose the recommended, easy, installation.

#### Start nginx
Passenger will take about five to ten minutes to install, configure, and optimize nginx with Ruby on Rails.

After it finishes, it will let you know about changes made to the nginx configuration file and how to deploy a Ruby on Rails application on your virtual server.

The last step is to turn start nginx, as it does not do so automatically.

```
sudo service nginx start 
```

nginx is now on. You can see the exciting “Welcome to nginx” screen in your browser if you point it toward http://youripaddress/

#### Connect Nginx to Your Rails Project

Once you have rails installed, open up the nginx config file

```
sudo vi /etc/nginx/conf/nginx.conf
```
Set the root to the public directory of your new rails project.

Your config should then look something like this:

```
server { 
listen 80; 
server_name example.com; 
passenger_enabled on; 
root /www/myapp/public; 
}
```

Install NodeJs if you do not yet have it:
```
sudo apt-get install nodejs
```

#### Installing PostgreSQL
```
sudo apt-get install postgresql postgresql-contrib libpq-dev
```

After postgreSQL is installted, create a production database and its user:
```
sudo -u postgres createuser -s myapp
```

Set the user’s password from psql console:
```
sudo -u postgres psql
```

After logging into the console, change the password:
```
postgres=# \password myapp
```

Enter your new password and confirm it. Exit the console with \q. It’s time to create a database for our application:
```
sudo -u postgres createdb -O myapp myapp_production
```

#### Configuration files
```
sudo mkdir -p /www/myapp/shared/config
vi /var/www/myapp/shared/config/database.yml
````

Paste the following in database.yml:
```
production:
  adapter: postgresql
  encoding: unicode
  database: myapp_production
  username: myapp
  password: myapp
  host: localhost
  port: 5432

```

After that, create secrets.yml
```
vi /var/www/myapp/shared/config/secrets.yml
```
and add the following:
```
SECRET_KEY_BASE: "70c90ed6d8912f3ab8a8bc1e3da0935377c2617cba19ccbe360f0b1e0aabd8809c0cdca6eb2ff8e070f45138c623cbabebeb335a03791d426df7be6f8b7bfd5e"
```

Change the secret to a new secret using the rake secret command.

We’re almost done with the server. Go back to local machine to start deployment with Capistrano.
```
cap production deploy
```

Since this is the first deployment, Capistrano will create all the necessary directories and files on the server, which may take some time. Capistrano will deploy the application, migrate the database, and start the application server. Now, login to the server and restart nginx so that our new configuration will reloaded:

```
sudo service nginx restart
```

Open up the browser and point it to ip. The application should be working now :)
