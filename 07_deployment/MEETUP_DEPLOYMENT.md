# Phase 4, Lecture 7 - Deployment

- Postgres
- Application environments and environment variables
- Production ready configuration
- Configuration of build steps for react application
- Telling Heroku how to run our application
- Setting up post release scripts that ensure our application will be able to respond to requests in production

Curriculum Resources:

- [Deploying a Rails API to Heroku](https://github.com/learn-co-curriculum/phase-4-deploying-rails-api-to-heroku)
- [Deploying Rails/React to Heroku](https://github.com/learn-co-curriculum/phase-4-deploying-rails-react-to-heroku)

I'm going to come at this in two phases. First, we're going to update our dependencies and configuration in preparation to deploy, then we're going to start working with the Heroku CLI to iterate on deployments. The goal will be to think about our production environment and make as many of the necessary adjustments as we can before we start attempting deployments. Because each deployment requires another commit, we want to try and anticipate as many of the required tasks as possible before we make the commit and do the push.

### Ruby Version

You can have any of the following Ruby versions as of July 2022: 2.7.6, 3.0.4, or 3.1.2. We've recommended that you install 2.7.4. Whichever version you've got, you'll need to run commands that look like these:

```
rvm install 2.7.6 --default
```

```
gem install bundler
gem install rails -v 6.1.6
```

If you don't have one of these versions installed at the moment, open up a terminal and start it now, it sometimes can take 10-15 minutes, but you'll be able to do some other stuff in the meantime.

For the meetup clone, we want to update the ruby-version to be 2.7.6 from our currently specified version of 2.7.4


## Switching to Postgres

Checking for local installation:

```
which postgres
```

or 

```
postgres --version
```

Sometimes I've seen students have [issues with Postgres on WSL](https://dakotaleemartinez.com/tutorials/postgresql-setup-on-ubuntu/) due to an older version that's installed but not active. In the lessons, the idea is presented that we should work with Postgres locally as well if we plan to use it in production. This is definitely a best practice, as it can help use prevent strange bugs from occuring down the line. But, if you are currently having issues with Postgres locally, you can most likely still deploy your application to production while using sqlite as the database to support your application in development. At scale, this is not practical as larger codebases are more likely taking advantage of features that postgres has that sqlite doesn't support, but at this point, you probably won't have any issues using Sqlite locally and Postgres in production. 

I'll go through how to configure your app for postgres locally and then also how to configure it to continue using the sqlite database we've used in development so you can see how to do that if you've been unable to resolve local issues with Postgres at this time.
### A note about Heroku

Heroku will automatically provision a new Postgres database with all of the adapter and connection information required for supporting a rails application and store all of that info in an environment variable called `DATABASE_URL`. When you create an application on Heroku for your project using the `heroku create` command (provided of the heroku CLI), a `DATABASE_URL` environment variable will be created for you containing all of the following configuration options:

- adapter
- database
- username
- password
- host
- port

This means that you won't need to set any of these values for the production environment within the `database.yml` file as rails will automatically assign those when it reads the value of the `DATABASE_URL` environment variable generated by Heroku.

Here is the `config/database.yml` file from the demo deployment app in the curriculum:

```yml
# PostgreSQL. Versions 9.3 and up are supported.
#
# Install the pg driver:
#   gem install pg
# On macOS with Homebrew:
#   gem install pg -- --with-pg-config=/usr/local/bin/pg_config
# On macOS with MacPorts:
#   gem install pg -- --with-pg-config=/opt/local/lib/postgresql84/bin/pg_config
# On Windows:
#   gem install pg
#       Choose the win32 build.
#       Install PostgreSQL and put its /bin directory on your path.
#
# Configure Using Gemfile
# gem 'pg'
#
default: &default
  adapter: postgresql
  encoding: unicode
  # For details on connection pooling, see Rails configuration guide
  # https://guides.rubyonrails.org/configuring.html#database-pooling
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: phase_4_deploying_demo_app_development

  # The specified database role being used to connect to postgres.
  # To create additional roles in postgres see `$ createuser --help`.
  # When left blank, postgres will use the default role. This is
  # the same name as the operating system user running Rails.
  #username: phase_4_deploying_demo_app

  # The password associated with the postgres role (username).
  #password:

  # Connect on a TCP socket. Omitted by default since the client uses a
  # domain socket that doesn't need configuration. Windows does not have
  # domain sockets, so uncomment these lines.
  #host: localhost

  # The TCP port the server listens on. Defaults to 5432.
  # If your server runs on a different port number, change accordingly.
  #port: 5432

  # Schema search path. The server defaults to $user,public
  #schema_search_path: myapp,sharedapp,public

  # Minimum log levels, in increasing order:
  #   debug5, debug4, debug3, debug2, debug1,
  #   log, notice, warning, error, fatal, and panic
  # Defaults to warning.
  #min_messages: notice

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  <<: *default
  database: phase_4_deploying_demo_app_test

# As with config/credentials.yml, you never want to store sensitive information,
# like your database password, in your source code. If your source code is
# ever seen by anyone, they now have access to your database.
#
# Instead, provide the password or a full connection URL as an environment
# variable when you boot the app. For example:
#
#   DATABASE_URL="postgres://myuser:mypass@localhost/somedatabase"
#
# If the connection URL is provided in the special DATABASE_URL environment
# variable, Rails will automatically merge its configuration values on top of
# the values provided in this file. Alternatively, you can specify a connection
# URL environment variable explicitly:
#
#   production:
#     url: <%= ENV['MY_APP_DATABASE_URL'] %>
#
# Read https://guides.rubyonrails.org/configuring.html#configuring-a-database
# for a full overview on how database connection configuration can be specified.
#
production:
  <<: *default
  database: phase_4_deploying_demo_app_production
  username: phase_4_deploying_demo_app
  password: <%= ENV['PHASE_4_DEPLOYING_DEMO_APP_DATABASE_PASSWORD'] %>
```

Note that there are some options set for the production environment here. They'll actually be ignored on Heroku because of what we discussed before. These values would only be used if we were to run a command like `RAILS_ENV=production rails db:create db:migrate` to create the production database on your local machine (where the DATABASE_URL configured by heroku won't be defined)

## Adding Postgres to the Meetup Clone

Here's what our `database.yml` file currently looks like for the meetup_clone application:

```yml
# SQLite. Versions 3.8.0 and up are supported.
#   gem install sqlite3
#
#   Ensure the SQLite 3 gem is defined in your Gemfile
#   gem 'sqlite3'
#
default: &default
  adapter: sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000

development:
  <<: *default
  database: db/development.sqlite3

# Warning: The database defined as "test" will be erased and
# re-generated from your development database when you run "rake".
# Do not set this db to the same as development or production.
test:
  <<: *default
  database: db/test.sqlite3

production:
  <<: *default
  database: db/production.sqlite3
```

All of this is currently configured for sqlite3. Assuming that we have Postgres installed properly, we'll need to add the `pg` gem to our Gemfile:

```
bundle add pg
```

This will install the gem locally add something like this to your Gemfile:

```rb
gem "pg", "~> 1.4"
```

If you've had issues with Postgres locally and are planning on keeping sqlite for the dev environment, then you'll want to do the following 2 steps:
1. Add a production group and wrap this line in it:

```rb
group :production do 
  gem "pg", "~> 1.4"
end
```
2. Move the sqlite gem to the :development, :test group
```rb
group :development, :test do
  # Call 'byebug' anywhere in the code to stop execution and get a debugger console
  gem 'byebug', platforms: [:mri, :mingw, :x64_mingw]
  # Use sqlite3 as the database for Active Record
  gem 'sqlite3', '~> 1.4'
end
```

If you're going ahead with full on Postgres, you'll want to remove the sqlite gem from the gemfile altogether. (I'm going to comment it out in my case so I can demonstrate both approaches)

> **NOTE**: In either case, it is important that the `sqlite3` gem not be in the default (ungrouped) part of the Gemfile as Heroku will complain if it is. (You can move `sqlite3` into the development, test group and Heroku won't mind as it only installs gems in the default and production groups)

Finally, when we deploy the application to Heroku, it'll be running on an Ubuntu machine, so we'll need to add a couple of platforms to the `Gemfile.lock` so Heroku will let the deploy go through.

```
bundle lock --add-platform x86_64-linux --add-platform ruby
```

Once the dependencies portion is complete, we can move to updating our database configuration so we can tell rails we want to use postgres in development.

## Configuring Rails for Postgres

First, we need to update the `config/database.yml` file.
```yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: 042522_meetup_clone_demo_app_development

test:
  <<: *default
  database: 042522_meetup_clone_demo_app_test

production:
  <<: *default
```

or, if you're sticking with sqlite3 in development, something like this:

```yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  adapter: sqlite3
  database: db/development.sqlite3

test:
  <<: *default
  database: 042522_meetup_clone_demo_app_test

production:
  <<: *default
```

After you've got the configuration updated here, we'll also need to create a new development database using postgres. Using sqlite, all we had to do was `rails db:migrate`. With Postgres, we'll need to run `rails db:create` first.

```bash
rails db:create db:migrate db:seed
```

If we get no printout here, that means we're good to go! When we were using Sqlite, we could use the Sqlite explorer to view information in our database right from within VSCode. We can add a [Postgres extension for VSCode](https://marketplace.visualstudio.com/items?itemName=ckolkman.vscode-postgres) to enable the same thing for Postgres. I recorded a quick demo of how to get started with the extension and make your first database connection. After you're set up initially, it's pretty similar to the SQLite Explorer.

<iframe width="560" height="315" src="https://www.youtube.com/embed/Cc9d2c8UuKA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Break
___

## Server vs Client Side Routing
We also need to handle our routing configuration. Requests coming from our react client to the API should be routed to our API endpoints, but when we click react router links, we want the app to still work upon page refresh. For this, we need to set up a fallback route that will catch 404s and render the `public/index.html` file

```rb
get "*path", to: "fallback#index", constraints: ->(req) { !req.xhr? && req.format.html? }
```


We're adding a couple of constraints to this route. This route will apply if we're not sending a fetch request and we're sending a request for html. This will mainly apply to the first time we visit the application and if we refresh the page any time thereafter. 

We also need to create the fallback controller and its index action.

```
rails g controller fallback
```

```rb
class FallbackController < ActionController::Base

  def index
    render file: 'public/index.html'
  end
end
```

## Removing conflict between client & server side routes

We need to create an api namespace for routes so that they all start with `/api`. For example: `/api/login`, `/api/groups` etc.

To do this, we need to do 3 things:

- Create a namespace within the `config/routes.rb` file and move all but the fallback route into it:
- create a folder called `api` at `app/controllers/api` and move the `EventsController`, `UsersController`, `SessionsController`, `MembershipsController`, `RsvpsController`, and `GroupsController` files into the `api` folder
- add `Api::` before each of the class names defined in the controller files listed in the previous step

#### Routes

```rb
# config/routes.rb
Rails.application.routes.draw do
  namespace :api do 
    resources :rsvps, only: [:create, :update, :destroy]
    resources :memberships, only: [:create, :destroy]
    resources :events
    resources :groups, only: [:index, :show, :create]
    resources :users
    # For details on the DSL available within this file, see https://guides.rubyonrails.org/routing.html
  
    get "/me", to: "users#show"
    post "/signup", to: "users#create"
    post "/login", to: "sessions#create"
    delete "/logout", to: "sessions#destroy"
  end

  get "*path", to: "fallback#index", constraints: ->(req) { !req.xhr? && req.format.html? }
end
```

#### An example of a namespaced controller

```rb
# app/controllers/api/sessions_controller.rb
class Api::SessionsController < ApplicationController
  skip_before_action :authenticate_user 

  # POST '/login'
  def create
    user = User.find_by(username: params[:username])
    if user&.authenticate(params[:password])
      session[:user_id] = user.id
      render json: user, status: :ok
    else
      render json: { errors: 'Invalid credentials' }, status: :unauthorized
    end
  end

  # DELETE '/logout'
  def destroy
    if current_user
      session.clear
    else
      render json: { errors: "No active session" }, status: :unauthorized
    end
  end


end
```

## Setting up the Client

To configure our client side application for building and deployment, we'll need to create a `package.json` file at the root of the project.

```bash
touch `package.json`
```

```json
{
  "name": "meetup_clone_client",
  "version": "1.0.0",
  "description": "Build and Deployment Configuration for Meetup Clone's React client",
  "engines": {
    "node": "16.13.2"
  },
  "scripts": {
    "clean": "rm -rf public",
    "build": "npm install --prefix client && npm run build --prefix client",
    "deploy": "cp -a client/build/. public/",
    "heroku-postbuild": "npm run clean && npm run build && npm run deploy"
  },
  "author": "DakotaLMartinez"
}
```

> NOTE: Make sure that your value for engines.node is the version of node that you see when you run `node -v` on your machine.

We now need to get our react client code into the same directory as our server side code:

```bash
git clone git@github.com:DakotaLMartinez/042522_meetup_clone_client.git client
```

Then let's remove the nested git repository from the cloned client:

```bash
rm -rf client/.git
```

## Update fetch requests to use /api namespaced routes

Replace all occurrences of:

``` 
fetch('
```

with 

```
fetch('/api)
```

Then replace all occurrences of:

```
fetch("
```

with 

```
fetch("/api
```

(This time be careful to only change files within the client directory!)

Finally, replace all occurrences of:

```
fetch(`
```

with

```
fetch(`/api
```

## Testing it out locally


When we get into our heroku configuration, we'll tell Heroku that there is a node component to our application. This will tell heroku to run the heroku-postbuild script after the application is deployed. When this happens, the public directory will be removed, we'll get an npm install and a build in the client directory and all of the contents of the build directory in client will be copied into the public directory of our rails application.

The fallback route that we defined in the previous step will render the public/index.html file that is created by this step.

At this point, we can test out this configuration by running: 
```
npm run heroku-postbuild
```
from our rails directory and then
```
rails s
```

If we visit our react app in the browser and navigate around and then refresh the page and everything works then we're on the right track.

Let's try it out, log in to the application and refresh the page.

-------
-------
-------
-------
-------
-------
-------
-------
-------
-------
-------
-------


## Configuring Heroku via the Heroku CLI

First, we want to create a Procfile in the root of the project that describes how to run our application.

```
touch Procfile
```
and add this to it:
```
web: bundle exec rails s
release: bin/rake db:migrate
```

Second, we need to make sure
Next, you'll want to make sure you're connected to your Heroku account 
```
heroku login
```
And then you'll be redirected to the browser. Once you sign into your account and click the button, you can return to the terminal and you should be logged in. 

Once you're logged in, you can create an application on heroku that you can deploy to.

```
heroku create
```

You'll see something like this if all is well:

```
Creating app... done, ⬢ stormy-basin-85902
https://stormy-basin-85902.herokuapp.com/ | https://git.heroku.com/stormy-basin-85902.git
```

And this adds in a remote that you can push to that will trigger a deployment. The syntax looks like this:

```
git push heroku main
```

If you want to push up the main branch. 

Before we actually run this, we'll want to tell heroku that we have a node and ruby application and that we want to build the nodejs part first (this will build the app and copy it to the public directory). We can configure this via the CLI using the following commands.

```
heroku buildpacks:add heroku/nodejs --index 1
heroku buildpacks:add heroku/ruby --index 2
```

When you've done both of these things, you should see something like this:

```
Buildpack added. Next release on stormy-basin-85902 will use:
  1. heroku/nodejs
  2. heroku/ruby
```
Since we're going to deploy via a git push, we need to commit all of our changes before we can do the deployment.

```
git add .
git commit -m "configure for heroku"
```

Heroku will only deploy from main or master, so we're going to push the master branch up:

```
git push heroku main
```

This should take quite a while to build, deploy, install and migrate the first time, but once we're done, we can run 

```
heroku open
```
from the CLI to check out the app in the browser. The first time I did this, I got an application error. It notifies you to check:

```
heroku logs --tail
```
to see what the problem was. After scrolling through the logs, I found this message:

2021-10-05T05:57:12.258256+00:00 app[web.1]: Exiting
2021-10-05T05:57:12.258461+00:00 app[web.1]: /app/vendor/bundle/ruby/2.7.0/gems/zeitwerk-2.4.2/lib/zeitwerk/loader/callbacks.rb:18:in `on_file_autoloaded': expected file /app/app/serializers/user_serializer.rb to define constant UserSerializer, but didn't (Zeitwerk::NameError)

So, I checked out that file and found this:

```rb
class ChangedUserSerializer < ActiveModel::Serializer
  attributes :id, :username, :email
end
```
Sure enough! We need to rename the class.

```rb
class UserSerializer < ActiveModel::Serializer
  attributes :id, :username, :email
end
```

then we need to commit the change
```
git add .
git commit -m "rename serializer class"
```

Then we can try the deploy again:

```
git push heroku master
```

That should do it!!!

# Heroku Deployment Cheat Sheet
## Dependencies (Gems/packages)
Add
```rb
gem "pg", "~> 1.2"
```
Remove 
```rb
gem 'sqlite3', '~> 1.4'
```
Add support for Ubuntu Linux
```
bundle lock --add-platform x86_64-linux --add-platform ruby
```
Get supported version of ruby (2.6.8 and 3.0.2 also work as of October 2021)
```
rvm install 2.7.4 --default
gem install bundler
gem install rails
```
## Configuration (environment variables/other stuff in config folder)

config/database.yml

```yml
default: &default
  adapter: postgresql
  encoding: unicode
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>

development:
  <<: *default
  database: meetup_clone_demo_app_development

test:
  <<: *default
  database: meetup_clone_demo_app_test

production:
  <<: *default
```
Procfile
```
web: bundle exec rails s
release: bin/rake db:migrate
```

package.json
```json
{
  "name": "meetup_clone_client",
  "version": "1.0.0",
  "description": "Build and Deployment Configuration for Meetup Clone's React client",
  "engines": {
    "node": ">= 14"
  },
  "scripts": {
    "clean": "rm -rf public",
    "build": "npm install --prefix client && npm run build --prefix client",
    "deploy": "cp -a client/build/. public/",
    "heroku-postbuild": "npm run clean && npm run build && npm run deploy"
  },
  "author": "DakotaLMartinez"
}
```
commands
```
heroku create
```

```
heroku buildpacks:add heroku/nodejs --index 1
heroku buildpacks:add heroku/ruby --index 2
```
## Database

No changes other than the switch to Postgres.

## Models

No changes
## Views

No changes (heroku postbuild will copy the react app into the rails public directory)

## Controllers

Fallback controller
```rb
class FallbackController < ApplicationController
  include ActionController::MimeResponds

  def index
    respond_to do |format|
      format.html { render body: Rails.root.join('public/index.html').read }
    end
  end
end
```

Ensure api endpoints are namespaced under Api and stored in a directory called app/controllers/api

## Routes

```rb
get "*path", to: "fallback#index", constraints: ->(req) { !req.xhr? && req.format.html? }
```

Ensure that api endpoints are namespaced under api so that they don't conflict with client side routes.

## Usage

To deploy the application you can push the main or master branch to the heroku remote using a command like this:

```
git push heroku main
```
or

```
git push heroku master
```
