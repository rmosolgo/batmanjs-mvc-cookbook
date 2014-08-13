# Batman.js & Ruby on Rails
\label{cha:batman_rails}


## Routing
\label{sec:rails_routing}

### Move Batman.js to Namespaced Routes
\label{sec:rails_namespaced_routes}

By default, batman-rails creates routes to serve the app from `/`, the root directory. You can serve the app from a namespaced route (eg, `/batman`) by updating the Rails configuration and the batman.js configuration.

First, modify the route created by batman-rails to include your namespace:

```ruby
  # add /#{namespace}/ here
  get "/batman/(*redirect_path)", to: "batman#index", constraints: lambda { |request| request.format == "text/html" }
```

Then, tell batman.js to use that path when generating routes:

```coffeescript
# declare configuration right before your app definition
Batman.config.pathToApp = "/batman/"

class MyApp extends Batman.App
  # ...
```

Now, batman.js will use `/batman/` for app routes.

