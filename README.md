# <img src="https://cloud.githubusercontent.com/assets/7833470/10899314/63829980-8188-11e5-8cdd-4ded5bcb6e36.png" height="60"> Rangular

| Objectives |
| :--- |
| Connect your client-side Angular application to a Rails server |

The great thing about Angular is that it's back-end-agnostic. Since Angular was built with CRUD in mind, as long as your Angular app can query RESTful API endpoints, it doesn't matter the stack of the server. As you've already seen, you don't even need to have your own server to get your Angular app working.

If you're going to build your own server from scratch to connect to your Angular app, you have many options, we've already looked at Mongo/Express/Node (MEN) and now we'll look at using Ruby on Rails with Postgres.

## Rails Angular Setup

#### Base Application

1. Create a new Rails application with a Postgres database:

  ```zsh
  ➜  rails new rails_angular_sample -Td postgresql
  ➜  cd rails_angular_sample
  ➜  rake db:create
  ```

2. Remove `turbolinks` from your Rails app (`Gemfile`, `application.js`, and `application.html.erb`(remove `'data-turbolinks-track' => true`)).

3. `generate` a `SiteController` with an `index` action. You'll also need to create `site/index.html.erb` inside `app/views`. Your `site#index` will serve as the "layout" for your Angular app.

#### Server Routes

1. Since `site#index` is the "layout" for your Angular app, you want the server to respond with this view every time a route is requested. This will allow Angular to handle routing on the client-side.

  You can use `get '*path'` to send every server-requested route to `site#index`:

  ```ruby
  #
  # config/routes.rb
  #

  Rails.application.routes.draw do
    root 'site#index'
    get '*path', to: 'site#index'
  end
  ```

#### Requiring Angular

1. As you've seen before, there are many ways to require assets in Rails. If you'd like, you can require the CDNs for Angular and `ngRoute` directly in the application layout:

  ```html
  <!-- app/views/layouts/application.html.erb -->

  <!DOCTYPE html>
  <html>
  <head>
    <%= csrf_meta_tags %>
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <base href="/">

  	<%= stylesheet_link_tag :application, media: :all %>

    <!-- angular -->
    <script type="text/javascript" src="https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.5.5/angular.min.js"></script>

    <!-- ngRoute -->
    <script type="text/javascript" src="https://ajax.googleapis.com/ajax/libs/angularjs/1.5.5/angular-route.min.js"></script>

    <%= javascript_include_tag :application %>

    <title>RailsAngularSample</title>
  </head>
  <body>
  	<%= yield %>
  </body>
  </html>
  ```

2. Another option is to download the files for Angular and `ngRoute` and include them in the asset pipeline.

  To download via CURL in the Terminal:

  ```zsh
  ➜  curl https://cdnjs.cloudflare.com/ajax/libs/angular.js/1.4.8/angular.min.js > vendor/assets/javascripts/angular.min.js
  ➜  curl https://ajax.googleapis.com/ajax/libs/angularjs/1.4.8/angular-route.min.js > vendor/assets/javascripts/angular-route.min.js
  ```

  And require in `application.js`:

  ```js
  /*
   * app/assets/javascripts/application.js
   */

  //= require angular.min
  //= require angular-route.min
  ```

#### Configuring Your Angular App

1. Create a new JavaScript file `app/assets/javascripts/app.js`. This is where you'll put all the logic for your Angular app.

2. Add the `ng-app` directive in the `<html>` tag in the application layout:

  ```html
  <!-- app/views/layouts/application.html.erb -->

  <!DOCTYPE html>
  <html ng-app="sampleApp">
  ...
  ```

3. Add the `ng-view` directive inside the `<body>` tag in `app/views/site/index.html.erb`:

  ```html
  <!-- app/views/site/index.html.erb -->

  <body>
    <nav class="navbar navbar-default">
      ...
    </nav>
    <div ng-view></div>
  </body>
  ```

  **Note:** Since this file serves as the "layout" for your Angular app, it's a good idea to place any shared code here, like a navbar.

4. Configure your Angular app in `app.js`:

  ```js
  /*
   * app/assets/javascripts/app.js
   */

  angular.module('sampleApp', ['ngRoute']);
  ```

#### Adding Templates

1. You can use the <a href="https://github.com/pitr/angular-rails-templates">angular-rails-templates</a> gem to add your Angular templates to the Rails asset pipeline.

  Require the gem in your `Gemfile`:

  ```ruby
  #
  # Gemfile
  #

  gem 'angular-rails-templates'
  ```

2. Require `angular-rails-templates` in `application.js`, as well as the path to your Angular templates (which you'll create in the next step):

  ```js
  /*
   * app/assets/javascripts/application.js
   */

  ...

  //= require angular-rails-templates
  //= require_tree ../templates
  ```

  **Note:** Make sure you require your templates AFTER your Angular files and BEFORE `app.js`.

3. Make a `templates` directory inside `app/assets`, and create a template:

  ```
  ➜  mkdir app/assets/templates
  ➜  touch app/assets/templates/home.html
  ```

  At this point, you should `bundle install` and restart your Rails server if you haven't already.

4. Add the `templates` module to your Angular app's dependencies in `app.js`:

  ```js
  /*
   * app/assets/javascripts/app.js
   */

  angular.module('sampleApp', ['ngRoute', 'templates']);
  ```

#### Configuring Angular Routes

1. Configure your Angular routes in `app.js` to hook everything up:

  ```js
  /*
   * app/assets/javascripts/app.js
   */

    angular.module('sampleApp', ['ngRoute', 'templates'])
        .config(config);

    config.$inject = ['$routeProvider', '$locationProvider'];
    function config (  $routeProvider,   $locationProvider  )  {
      $routeProvider
        .when('/', {
          templateUrl: 'home.html',
          controller: 'HomeIndexController',
          controllerAs: 'homeIndexCtrl'
        })
        .otherwise({
          redirectTo: '/'
        });

      $locationProvider
        .html5Mode({
          enabled: true,
          requireBase: false
        });
    };
  ```

2. Configure your controller with some test data, so you can check to see if the route, template, and controller are properly connected:

  ```js
  /*
   * app/assets/javascripts/app.js
   */

  ...
  .controller('HomeIndexController', HomeIndexController);

  HomeIndexController.$inject=[];
  function HomeIndexController() {
    var vm = this;
    vm.greeting = "what's up?"
  };

  ```
3. Test `greeting` in your view.

#### Miscellaneous Setup

1. Since Rails has extra security around submitting forms, there's some setup involved in the `ApplicationController` to allow your Angular app to submit data to the server:

  ```ruby
  #
  # app/controllers/application_controller.rb
  #

  class ApplicationController < ActionController::Base
    # Prevent CSRF attacks by raising an exception.
    # For APIs, you may want to use :null_session instead.
    protect_from_forgery with: :exception

    after_filter :set_csrf_cookie_for_ng

    protected
    def verified_request?
      super || valid_authenticity_token?(session, request.headers['X-XSRF-TOKEN'])
    end

    private
    def set_csrf_cookie_for_ng
      cookies['XSRF-TOKEN'] = form_authenticity_token if protect_against_forgery?
    end
  end
  ```

#### CRUD

Now that your Angular app is all set up, it's time to CRUD a resource! You'll need:

1. CRUD routes for your resource:

  ```ruby
  #
  # config/routes.rb
  #

  Rails.application.routes.draw do
    root 'site#index'

    namespace :api, defaults: { format: :json } do
      resources :todos, except: [:new, :edit]
    end

    get '*path', to: 'site#index'
  end
  ```

2. A controller with CRUD actions that renders JSON. Run `rails g controller api/todos`, then:

  ```ruby
  #
  # app/controllers/api/todos_controller.rb
  #

  class Api::TodosController < ApplicationController
    def index
      @todos = Todo.all.order("created_at DESC")
      render json: @todos
    end

    def create
      @todo = Todo.new(todo_params)
      if @todo.save
        render json: @todo
      else
        render json: { errors: @todo.errors.full_messages.join(", ") }, status: :unprocessable_entity
      end
    end

    ...

    private
    def todo_params
      params.require(:todo).permit(:title, :description, :done)
    end

  end
  ```

3. Use the Angular `$http` service to interact with your JSON API endpoints. See the module on <a href="https://github.com/SF-WDI-LABS/shared_modules/tree/master/03-angular-mean/http/28">http</a> for reference.

## Challenges

1. Follow the steps above to set up a new Rangular application.

## Stretch Challenges

1. Once you app is set up, build a JSON API to CRUD one resource (`todos` are always a good start). You can test your API routes on Postman.

2. Use Angular `$http` to query your API endpoints from the client side to implement full CRUD in your single-page app.

3. Create a new show details route.

## Solution
* <a href="https://github.com/SF-WDI-LABS/rangular-todo" target="_blank">Rangular Todo</a>
