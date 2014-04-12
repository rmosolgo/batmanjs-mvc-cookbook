# Model
\label{cha:batman_model}

## Find a Record in the Memory Map
\label{sec:memory_map}
Find it by ID (returns `undefined` if not present):

```coffeescript
cachedItem = App.Product.get('loaded').indexedByUnique('id').get(productId)
```

Find it by ID, create a new record if not found (always returns a record):

```coffeescript
cachedItem = App.Product.createFromJSON({id: productId})
```

## Remove a Record from the Memory Map
\label{sec:remove_from_memory_map}

Each model's `loaded` set is a `Batman.Set`, so you can use its `remove` function.

```coffeescript
App.Product.get('loaded').remove(unwantedProduct)
```

## Load a Record from a Specific URL
\label{sec:load_from_url}

Set its `url` property and `load` it.

```coffeescript
specialProduct = new App.Product
specialProduct.url = "/products/special"
specialProduct.load()
```

## Use Specific URL for Storage
\label{sec:url_for_storage}

Use `Model.url` with `Batman.RestStorage` (or `Batman.RailsStorage`):

```coffeescript
class MyApp.Country extends Batman.Model
  @persist Batman.RestStorage
  @url: "/api/v1/countries"
```

## Asynchronous Accessor Value
\label{sec:async_accessor}

Use "promise accessor" syntax. The accessor function takes a `promise` key. The function should call `deliver` with the value. If it doesn't return `undefined`, the returned value will be considered an early return value.

```coffeescript
class MyApp.Person extends Batman.Model
  # asynchronously load all cousins
  @accessor 'allCousins',
    promise: (deliver) ->
      return undefined if @isNew()
      cousinURL = "#{App.Person.url}.json?cousin_id=#{@get('id')}"
      new Batman.Request
        url: cousinURL
        success: (response) ->
          cousins = App.Person.createMultipleFromJSON(response)
          deliver(null, cousins)
      undefined
```

## Delegate Model Attributes
\label{sec:delegate_attributes}

Use `Batman.Object.delegate`:

```coffeescript
  @delegate 'name', 'birthdate' to: 'person'
```

## Provide a Formatted Version of a Model Attribute
\label{sec:formatted_attribute}

Create a get-only accessor which returns the formatted value. Branching in `@accessor` bodies is very safe becuase the accessor tracks its sources.

```coffeescript
  @accessor 'createdAtFormatted', ->
    if @get('created_at').isSame(moment(), 'day')
      "Today at #{@get('created_at').format('h:mm')}"
    else if @get('created_at').isSame(moment().subtract(1, 'day') , 'day')
      "Yesterday at #{@get('created_at').format('h:mm')}"
    else
      "On #{@get('created_at').format(App.get('momentDateFormat'))} at #{@get('created_at').format('h:mm')}"
```

Or create an accessor that maps between stringified values and raw values:

```coffeescript
  # Get and set `mode` through a stringified version, `modeString`
  @accessor 'modeString',
    get: ->
      switch @get('mode')
        when 0 then "Free"
        when 1 then "Limited"
    set: (modeString, value) ->
      switch modeString
        when "Free" then @set('mode', 0)
        when "Limited" then @set('mode', 1)
```

## Validate Inclusion for a Multiple-Choice Attribute
\label{sec:validate_inclusion}

```coffeescript
class App.EmailAddress extends App.Model
  @LOCATIONS: ['Home', 'Work']
  @encode 'location'
  @validate 'location', inclusion: {in: @LOCATIONS}
```

## Create Boolean Accessors for Multiple-Choice Attributes
\label{sec:boolean_accessors}

Store values in a class-level "constant", then iterate over them to create accessors. Mind the fat arrows!

```coffeescript
class App.User extends Batman.Model
  @encode 'permission'
  @PERMISSIONS = ["Viewer", "Editor", Administrator"]

  @PERMISSIONS.forEach (permission) =>
    # create accessors like `isViewer`
    allPermissions = App.Person.PERMISSIONS
    permissionIndex = allPermissions.indexOf(permission)
    @accessor "is#{Batman.helpers.capitalize(permission)}", ->
      ownPermission = @get('permission')
      allPermissions.indexOf(ownPermission) is permissionIndex
```

Additionally, we could create App-wide `is#{Permission}` accessors for the current Person:

```coffeescript
# eg, App.get('isAdministrator')
App.Person.PERMISSIONS.forEach (permission) ->
  App.classAccessor "is#{permission}", -> App.get("Person.current.is#{permission}")
```

## Provide Custom Validation for Attribute
\label{sec:custom_validation_function}

Implement a function with this signature:

```coffeescript
  @validate 'location', (errors, record, key, callback) ->
    locationValid = @_checkValidity() # implemented elsewhere...
    if !locationValid
      errors.add("location", "is not valid")
    callback()
```

- add items to `errors` to show batman.js that validation failed
- call `callback` to continue the validation process.
- validations may be asynchronous

## Set Values Without Dirtying the Model
\label{sec:without_dirty_tracking}

Use the semi-private `Model::_withoutDirtyTracking` function:

```coffeescript
class App.Country extends Batman.Model
  setDefaults: ->
    @_withoutDirtyTracking ->
      @set 'government', -> 'Anarchy'
      @set 'name', -> ''
      @set 'founded_at', -> new Date
      if !@get('provinces')?
        @resetProvinces()
```

Nothing inside the passed function will dirty the record's attributes.

## Set Values Only If They're Undefined
\label{sec:get_or_set}

Use `Batman.Object::getOrSet`, which sets the value if it's `undefined`.

```
class App.PhoneNumber extends Batman.Model
  @encode 'number', 'location'
  ensureLocationPresent: ->
    @getOrSet('location', -> 'Home')
```