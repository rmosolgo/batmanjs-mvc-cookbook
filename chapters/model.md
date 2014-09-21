# Model
\label{cha:batman_model}

__To my knowledge, batman.js is currently unmaintained. For this reason, I don't recommend starting your next project with batman.js!__

## Working with Records
\label{sec:records}

### Find a Record in the Memory Map
\label{sec:memory_map}
Find it by ID (returns `undefined` if not present):

```coffeescript
cachedItem = App.Product.get('loaded').indexedByUnique('id').get(productId)
```

Find it by ID, create a new record if not found (always returns a record):

```coffeescript
cachedItem = App.Product.createFromJSON({id: productId})
```

### Remove a Record from the Memory Map
\label{sec:remove_from_memory_map}

Each model's `loaded` set is a `Batman.Set`, so you can use its `remove` function.

```coffeescript
App.Product.get('loaded').remove(unwantedProduct)
```

### Filter Records by Attribute Value

Use `indexedBy` to create a `Batman.SetIndex`, a set of records with a given value.

For example, to find all `Product`s in the "toys" category:

```coffeescript
toys = App.Product.get('all.indexedBy.category').get('toys')
```

If you need a more complex filter function, create an accessor on the model:

```coffeescript
class App.Product extends Batman.Model
  @accessor 'forChildren', ->
    @get('recommendedAge') <= 12 || @get('category') is 'toys'
```

Then, use that accessor to index the records:

```coffeescript
productsForChildren = App.Product.get('all.indexedBy.forChildren').get(true)
```

### Sort Records by Attribute Value

Use `sortedBy` to create a `Batman.SetSort`.

```coffeescript
productsByName = App.Product.get('all.sortedBy.name')
```

For a more complicated sort, define a new accessor and use it to sort the records:

```coffeescript
class App.Product extends Batman.Model
  @accessor 'salesPerYear', ->
    @get('totalSales') / @get('yearsOnTheMarket')
# ...
topSellers = App.Product.get('all.sortedByDescending.salesPerYear')
```

## Getting Data from Storage
\label{sec:storage}

### Load a Record from a Specific URL
\label{sec:load_from_url}

Set its `url` property and `load` it.

```coffeescript
specialProduct = new App.Product
specialProduct.url = "/products/special"
specialProduct.load()
```

### Load a Collection from a Specific URL
\label{sec:load_collection_from_url}

You can specify a custom `url` at load-time by passing it as an option to `Model.load`. For example:

```coffeescript
App.Question.load {url: "/questions/latest"}, (err, records) ->
  # records is an array of loaded records
```

You can also:

- Set a base URL for the model (Section~\ref{sec:url_for_storage})
- Define nested URLs (Section~\ref{sec:nested_urls})

### Use Specific URL for Storage
\label{sec:url_for_storage}

Use `Model.url` with `Batman.RestStorage` (or `Batman.RailsStorage`):

```coffeescript
class MyApp.Country extends Batman.Model
  @persist Batman.RestStorage
  @url: "/api/v1/countries"
```

### Use Nested URLs for Loading Records
\label{sec:nested_urls}
When using REST storage, you can call `@urlNestsUnder` in the model definition. If any of the "parents" are available at load-time, they'll be used to generate a URL.

For example, you can define possible parents "category" and "user":

```coffeescript
class App.Question extends Batman.Model
  @persist Batman.RestStorage # or any subclass of RestStorage
  @urlNestsUnder 'category', 'user'
```

Then load questions for a category:

```coffeescript
App.Question.load {category_id: 43}, (err, records) -> # will use /categories/43/questions
```

Or load questions for a user:

```coffeescript
App.Question.load {user_id: 901}, (err, records) -> # will use /users/901/questions
```

Or, simply load all questions:

```coffeescript
App.Question.load (err, records) -> # will use base URL
```






### Reload All Records
\label{sec:reload_all_records}

Use `Model.load` to force a reload from the storage adapter.

```coffeescript
MyApp.Country.load() # will initiate a request
```

This will add new records, but __won't__ remove destroyed records. To remove destroyed records, replace the `loaded` set with the loaded records:

```coffeescript
MyApp.Country.load (err, records) ->
  throw err if err?
  recordSet = new Batman.Set(records...)
  MyApp.Country.get('loaded').replace(recordSet)
```

Using `replace` means batman.js will automatically update all bindings and accessors with any added or removed items.

### Get a Field from Storage, but Don't Send It Back
\label{sec:one_way_encoding}

Pass an object to `@encode` whose `encode` property is `false`.

For example, to get `total_population` from storage, but not send it back when saving, use:

```coffeescript
class MyApp.Country extends Batman.App
  @encode 'total_population',
    encode: false
    # the default decoder will be used
```

Now, `total_population` won't be present in the JSON sent back to the server.

## Attributes
\label{sec:attributes}

### Asynchronous Accessor Value
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

### Delegate Model Attributes
\label{sec:delegate_attributes}

Use `Batman.Object.delegate`:

```coffeescript
  @delegate 'name', 'birthdate' to: 'person'
```

### Provide a Formatted Version of a Model Attribute
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

### Validate Inclusion for a Multiple-Choice Attribute
\label{sec:validate_inclusion}

```coffeescript
class App.EmailAddress extends App.Model
  @LOCATIONS: ['Home', 'Work']
  @encode 'location'
  @validate 'location', inclusion: {in: @LOCATIONS}
```

### Create Boolean Accessors for Multiple-Choice Attributes
\label{sec:boolean_accessors}

Store values in a class-level "constant", then iterate over them to create accessors. Mind the fat arrows!

```coffeescript
class App.User extends Batman.Model
  @encode 'permission'
  @PERMISSIONS = ["Viewer", "Editor", "Administrator"]

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

### Provide Custom Validation for Attribute
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

### Set Values Without Dirtying the Model
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

### Set Values Only If They're Undefined
\label{sec:get_or_set}

Use `Batman.Object::getOrSet`, which sets the value if it's `undefined`.

```coffeescript
class App.PhoneNumber extends Batman.Model
  @encode 'number', 'location'
  ensureLocationPresent: ->
    @getOrSet('location', -> 'Home')
```

### Provide Default Values
\label{sec:default_values}

In the constructor, set values for undefined keys (Section~\ref{sec:get_or_set}) without dirtying the record (Section~\ref{sec:without_dirty_tracking}).

For example, to default `PhoneNumber::location` to `"Home"`:

```coffeescript
class App.PhoneNumber extends Batman.Model
  constructor: ->
    super
    @_withoutDirtyTracking ->
      @getOrSet 'location', -> "Home"
```

## Associations
\label{sec:associations}

Recipes will draw from these models:

```coffeescript
class App.Team extends Batman.Model
  @encode 'name'

class App.Player extends Batman.Model
  @encode 'name', 'position'
```

### Corresponding Has-Many and Belongs-To
\label{sec:has_many_belongs_to}
To set up "player belongs to team", declare a `belongsTo` association in the `Player` definition:

```coffeescript
class App.Player extends Batman.Model
  @belongsTo 'team'
```

To set up "team has many players", declare a `hasMany` in the `Team` definition:

```coffeescript
class App.Team extends Batman.Model
  @hasMany 'players', inverseOf: 'team'
```

_Be sure to include `inverseOf`, if possible. It sets up the 2-way association when loading from JSON._

### Has-Many Association with Inline JSON
\label{sec:has_many_inline}

Use `saveInline: true` in the association definition:

```coffeescript
class App.Team extends Batman.Model
  @hasMany 'players', inverseOf: 'team', saveInline: true
```

### Adding and Removing Has-Many Children in a Form
\label{sec:has_many_children_in_form}

__NOTE:__ until batman.js v0.17, you should define `namespace: MyApp` if you want to use `AssociationSet::build`. This bug has been fixed in master.

Use a custom view to encapsulate adding & removing. Then, connect the view to the form with `data-event` bindings.

The view could look like this:

```coffeescript
class App.TeamPlayersFormView extends Batman.View
  addPlayer: (team) ->
    # team.get('players').build() # see note about `build` in v0.16
    player = new App.Player(team: team)
    team.get('players').add(player)

  removePlayer: (team, player) ->
    team.get('players').remove(player)
    # if you aren't using saveInline: true, you may want to delete the player, too:
    # player.destroy()
```

Then, you can wire it up to your HTML:

```html
<div data-view='TeamPlayersFormView'>
  <a data-event-click="addPlayer | withArguments player">
    Add a Player
  </a>
  <ul>
    <li data-foreach-player='team.players'>
      <input type='text' data-bind='player.name' />
      <a data-event-click='removePlayer | withArguments team, player'>
        Remove this Player
      </a>
    </li>
  </ul>
</div>
```

### Unloading Associations on Destroy
\label{sec:unload_association_on_destroy}

It is often useful to remove associated models upon destroy. We can accomplish this by using the `destroyed` model lifecycle listener:

```coffeescript
class App.Team extends Batman.Model
  @hasMany 'players', inverseOf: 'team'

  @::on 'destroyed', -> @unloadAssociated('players')

  unloadAssociated: (label) ->
    @get(label).forEach (record) ->
      record.constructor.get('loaded').remove(record)
```



