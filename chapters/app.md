# App

## Put The App in the Global Scope
\label{sec:global_scope}

Use `@` in a coffeescript file to access `window`:

```coffeescript
class @MyApp extends Batman.App
```


## Run App When Page Is Ready
\label{sec:run}

Use jQuery to call `run` on `document.ready`:

```coffeescript
$ -> App.run()
```

(Sure, you could use pure JavaScript too but ... you've probably got jQuery already anyways!)

## Initialize App
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

## Handle the Root Path ("/")
\label{sec:root_path}

```coffeescript
  @root 'events#index'
```

## Define a Specific Route
\label{sec:specific_route}

If the controller action doesn't match a `@resources` path, use `@route`:

```coffeescript
  @route 'check_ins/last', 'checkIns#last'
```

## Handle Missing Routes
\label{sec:missing_routes}

Batman.js redirects all missing routes to `/404`

```coffeescript
  @route '/404', (params) -> App.showDialog("errors/show404")
```

## Create App-wide Accessors
\label{sec:app_accessors}

Use `@classAccessor` inside the App definition:

```coffeescript
  @classAccessor 'stateManager',
    get: -> @_manager ?= new App.StateManager
    set: (key, value) -> @_manager = value
    unset: (key) -> @_manager = undefined
```

## Create App-Wide Functions
\label{sec:app_functions}

Define class functions in the App defintion:

```coffeescript
  @goBack: -> history.back()
```

Since view bindings can see properties on `Batman.currentApp`, you can use app-wide functions in HTML, too:

```html
<a data-event-click="goBack">Back</a>
```