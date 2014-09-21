# Storage Adapter
\label{cha:storage_adapter}

__To my knowledge, batman.js is currently unmaintained. For this reason, I don't recommend starting your next project with batman.js!__

## Custom Storage Adapters
\label{sec:custom_storage_adapters}

### Define a Custom Storage Adapter
\label{sec:create_custom_storage_adapter}

You can create a custom storage adapter by extending one of batman.js's built-in storage adapters:

```coffeescript
class App.CustomStorage extends Batman.RestStorage
  # ... before/after actions, overriding methods
```

Or by extending `Batman.StorageAdapter` and implementing the required functions, `read`, `readAll`, `create`, `update`, and `destroy`:

```coffeescript
class App.CustomStorage extends Batman.RestStorage
  read: ->
  readAll: ->
  create: ->
  update: ->
  destroy: ->
```

See the [batman.js Storage Adapter docs](http://batmanjs.org/docs/api/batman.storageadapter.html) for more information on implementing the `StorageAdapter` interface.

### Use a Custom Storage Adapter to Persist Models
\label{sec:use_custom_storage_adapter}

After creating a custom storage adapter (Section~\ref{sec:create_custom_storage_adapter}), pass it to `@persist` in model definition to use it in that model:

```coffeescript
class App.Sandwich extends Batman.Model
  @persist App.CustomStorage
  @url: '/api/sandwiches'
```

Make sure that `App.CustomStorage` is loaded _before_ any models that use it. `@persist` is evaluated at load time, so if the storage adapter isn't defined yet, you'll get an error.

### Send a Custom Header with HTTP Requests
\label{sec:http_headers}

Extend `Batman.RestStorage` (or `Batman.RailsStorage`) and add a before-action to add to the headers:

```coffeescript
class App.CustomHeaderStorage extends Batman.RestStorage
  @::before 'all', (env, next) ->
    headers = env.options.headers ||= {}
    headers["App-Specific-Header"] = App.getSpecificHeader()
    next()
```

Then, use `App.CustomHeaderStorage` to persist your models (Section~\ref{sec:use_custom_storage_adapter})

### Append `.json` to All URLs
\label{sec:url_suffix}

This would allow URL configuration like:

```coffeescript
class App.Sandwich extends Batman.Model
  @url: '/api/sandwiches'
```

to become `/api/sandwiches.json` or `/api/sandwiches/1.json`.

This functionality is provided by `Batman.RailsStorage`, but if you don't want to use the `batman.rails` CoffeeScript extra, you could easily implement it in a custom storage adapter:

```coffeescript
class App.JSONStorage extends Batman.RestStorage
  # override the default URL functions to add .json:
  urlForRecord: -> @_addJsonExtension(super)
  urlForCollection: -> @_addJsonExtension(super)

  _addJsonExtension: (url) ->
    if url.indexOf('?') isnt -1 or url.substr(-5, 5) is '.json'
      return url
    url + '.json'
```

Then, use `App.JSONStorage` to persist your models (Section~\ref{sec:use_custom_storage_adapter})

## Batman.RestStorage

### Don't Send JSON with Namespace

The JSON spec includes namespacing all objects. However, if you don't want to do that, stub out the namespace function in `Batman.RestStorage`:

```coffeescript
Batman.RestStorage::recordJsonNamespace = -> false
```

### Send Data as JSON, Not Form Data

By default, `Batman.RestStorage` sends data like HTML form data. If you want to send JSON, pass `{serializeAsForm: false}` when passing the adapter to `@persist`:

```coffeescript
@persist Batman.RestStorage, serializeAsForm: false
```

