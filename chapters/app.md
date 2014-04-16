# App
\label{cha:batman_app}


## Setup
\label{sec:setup}

### Put the App in the Global Scope
\label{sec:global_scope}

Use `@` in a coffeescript file to access `window`:

```coffeescript
class @MyApp extends Batman.App
```

### Run the App When Page Is Ready
\label{sec:run}

Use jQuery to call `run` on `document.ready`:

```coffeescript
$ -> App.run()
```

(Sure, you could use pure JavaScript too but ... you've probably got jQuery already anyways!)

### Initialize the App
\label{sec:initialize}

Use the App's `run` event:

```coffeescript
class @App extends Batman.App
  @on 'run', ->
    @_applyMixins()
    App.set 'logger', new Logger()
    App.Person.find CURRENT_PERSON_ID, (err, record) ->
      App.Person.set('current', record)
    App.ClientReloader.get('instance').bind(App.pusherChannel)
```

### Compile the App (with Rails)
\label{sec:compile_with_rails}

Compiling batman.js with the Rails asset pipeline is a breeze. If you're using the batman-rails gem, `rails generate batman:app` will set everything up for you.

Use sprockets `require` directives to require dependencies when extending your own classes:

```coffeescript
#= require ./product
class App.GiftBasket extends App.Product
```

If you're not using batman-rails, use sprockets `require` directives to include whole directories in the main application file:


```coffeescript
#= require ./lib/batman
#= require ./lib/jquery
#= require_self
#= require_tree ./models
#= require_tree ./views
# ...
class @App extends Batman.App
```

### Compile the App (without Rails)
\label{sec:compile_without_rails}

Batman.js doesn't include a loader. You must compile the CoffeeScript and ensure proper load order yourself.

Your options include:

- [Gulp.js](http://gulpjs.com/): See [this blog post](http://rmosolgo.github.io/blog/2014/03/22/using-gulp-dot-js-to-build-batman-dot-js-without-rails/).
- [Harp.js](http://harpjs.com/): good for building front-end only.
- Please [add to this project](https://github.com/rmosolgo/batmanjs-mvc-cookbook) if you have another good solution!

## Routing
\label{sec:routing}

### Handle the Root Path ("/")
\label{sec:root_path}

```coffeescript
  @root 'events#index'
```

### Define a Specific Route
\label{sec:specific_route}

If the controller action doesn't match a `@resources` path, use `@route`:

```coffeescript
  @route 'check_ins/last', 'checkIns#last'
```

### Handle Missing Routes
\label{sec:missing_routes}

Batman.js redirects all missing routes to `/404`

```coffeescript
  @route '/404', (params) -> App.showDialog("errors/show404")
```

## Other
\label{sec:other}

### Create App-Wide Accessors
\label{sec:app_accessors}

Use `@classAccessor` inside the App definition:

```coffeescript
  @classAccessor 'stateManager',
    get: -> @_manager ?= new App.StateManager
    set: (key, value) -> @_manager = value
    unset: (key) -> @_manager = undefined
```

### Create App-Wide Functions
\label{sec:app_functions}

Define class functions in the App defintion:

```coffeescript
  @goBack: -> history.back()
```

Since view bindings can see properties on `Batman.currentApp`, you can use app-wide functions in HTML, too:

```html
<a data-event-click="goBack">Back</a>
```