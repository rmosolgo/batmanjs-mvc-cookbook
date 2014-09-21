# Principles: Batman.Object
\label{sec:object}

__To my knowledge, batman.js is currently unmaintained. For this reason, I don't recommend starting your next project with batman.js!__


`Batman.Object` is the superclass of (virtually) all objects in Batman.js, so these pointers apply to all the classes you'll interact with:

- `Batman.App`
- `Batman.Model`
- `Batman.Controller`
- `Batman.View`
- `Batman.Set` (as well as `SetIndex`, `SetSort`, `SetMapping`, etc)
- `Batman.Hash`

You can also extend `Batman.Object` to create your own objects!

## Property Basics
\label{sec:property_basics}

### Get, Set, Unset
\label{sec:get_and_set}

Every object has properties that are accessed with `get` and `set` (and `unset`).

The getters and setters (and unsetters) are simple to use:

```coffeescript
obj = new Batman.Object
obj.get('someProperty')     # => undefined
obj.set('someProperty', 12) # => 12
obj.get('someProperty')     # => 12
obj.unset('someProperty')
obj.get('someProperty')     # => undefined
```

When using `get` and `set`, you can:

- access properties defined with custom `@accessor` functions (as described below)
- access properties which use a "default accessor"

Whenever you `get` or `set` a property that doesn't have a custom `@accessor`, batman.js uses that object's default accessor.


### Properties & Accessors
\label{sec:properties_and_accessors}

In batman.js, "property" and "accessor" are closely related concepts.

Generally, _objects have properties_. When you interact with objects, you may get or set properties. _Properties have values_.

For example, we can access the `"flavor"` property and get its value, `"Vanilla"`:

```coffeescript
cake = new Batman.Object(flavor: "Vanilla", icingFlavor: "Chocolate")
cake.get('flavor') # => Vanilla
```

When defining a custom `Batman.Object`, you may _define accessors_. Accessors are the _functions_ used to `get` and `set` property values.

For example, we can define a `isAllChocolate` accessor on `Cake`:

```coffeescript
class Cake extends Batman.Object
  @accessor 'isAllChocolate', ->
    @get('flavor') is "Chocolate" and @get('icingFlavor') is "Chocolate"
```

After defining the _accessor_, we can get the value by accessing the _property_:

```coffeescript
cake = new Batman.Object(flavor: "Chocolate", icingFlavor: "Chocolate")
cake.get('isAllChocolate') # => true
```

### Keypaths
\label{sec:keypaths}

Properties are accessed _by name_ with `get`/`set`. In fact, you can access "deep" properties by using `.`. Since these property names can be _paths_ to _keys_ on other objects, they are sometimes called __keypaths__.

Consider the `city` object, whose `state` is another object:

```coffeescript
virginia = new Batman.Object(name: "Virginia")
city = new Batman.Object(name: "Charlottesville", state: virginia)
```

From the `city`, we can access its `"name"` and `"state"`:

```coffeescript
city.get("name")  # => "Charlottesville"
city.get("state") # => <#Batman.Object name="Virginia">
```

What if we wanted to access the _name_ of the `"state"`? We could perform two `get`s:

```coffeescript
city.get("state").get("name") # => "Virginia"
```

Or we could access a _nested property_, `"state.name"`:

```coffeescript
city.get("state.name") # => "Virginia"
```

`"state.name"` is a __keypath__.

Notice how `"state.name"` is a path to the `"name"` key on the `state` object. It's like a filepath: it's a pointer to some location which may hold a value.

View bindings take keypaths. Those keypaths lookup values in the view's render context.

Some pointers about keypaths:

- Keypath lookups are safe as long as you're looking up values on `Batman.Object`s. Inside accessors, they participate in source tracking just like using `get`.
- Keypaths may access properties on plain JavaScript objects, but they will be cached and they won't be updated if that JavaScript object changes. Be careful!
- A keypath is made of _segments_ by splitting on the `.`s. If any segment returns `undefined`, the lookup will return `undefined`.

### Accessor Inheritance
\label{sec:accessor_inheritance}

A subclass of `Smoothie` (above) would also have a `flavor` accessor which uses the logic defined in `Smoothie`. Later implementations of accessors overwrite earlier ones.

`Milkshake` is a subclass of `Smoothie`:
```coffeescript
class Milkshake extends Smoothie
```

Instances of `Milkshake` can use accessors defined on their parent class:

```coffeescript
milkshake = new Milkshake(hasCherries: true, hasMangos: true)
milkshake.get('flavor') # "cherry-mango"
```

## Properties Under the Hood
\label{sec:properties_under_the_hood}

### Source Tracking
\label{sec:source_tracking}

Batman.js properties track their sources. Consider this accessor:

```coffeescript
class Smoothie extends Batman.Object
  @accessor 'flavor', ->
    flavors = []
    flavors.push("strawberry") if @get('hasStrawberries')
    flavors.push("banana") if @get('hasBananas')
    flavors.push("cherry") if @get('hasCherries')
    flavors.push("mango") if @get('hasMangos')
    flavors.join("-")
```

When you get the flavor of a smoothie:

```coffeescript
bananaCherrySmoothie = new Smoothie(hasBananas: true, hasCherries: true)
bananaCherrySmoothie.get('flavor')
```

four `@get`s will be fired:

| `hasStrawberries` |
| `hasBananas` |
| `hasCherries` |
| `hasMangos` |

All four of those properties  will be registered internally as sources of `flavor`. If one of them is changed via `set`, the `flavor` accessor will be executed again and the new `flavor` value will be created.

### Caching
\label{sec:caching}

Batman.js caches property lookups. As described above, accessors are only recalculated when their sources change, not whenever they're fetched with `get`.


You can override batman.js's caching with `cache: false`. So, these accessors behave very differently:

```coffeescript
class NumberGenerator extends Batman.Object
  @accessor 'onceRandom', -> Math.rand()

  @accessor 'alwaysRandom',
    get: -> Math.rand()
    cache: false
```

`onceRandom` will only be calculated once, then cached:

```coffeescript
numberGenerator.get('onceRandom') # => 0.8543
numberGenerator.get('onceRandom') # => 0.8543
numberGenerator.get('onceRandom') # => 0.8543
```

but `alwaysRandom` will be reevaluated whenever it's accessed:

```coffeescript
numberGenerator.get('alwaysRandom') # => 0.2631
numberGenerator.get('alwaysRandom') # => 0.7328
numberGenerator.get('alwaysRandom') # => 0.9114
```

### Property, Accessor and Keypath Internals
\label{property_internals}

"Property", "Accessor" and "Keypath" have special meaning inside batman.js internals.

#### Batman.Property

`Batman.Property` is a class whose instances support batman.js's observable property system. You can access the underlying `Batman.Property` objects with `Batman.Object::property`:

```coffeescript
obj = new Batman.Object
obj.property('someProperty') # => <# Batman.Property >
```

_(In fact, these may be `Batman.Keypath` instances, as described below. `Batman.Keypath` extends `Batman.Property`.)_

Also, every `Batman.Object` has a `Batman.SimpleHash` filled with its `Batman.Property` instances. You can access these from a `Batman.Object`
s `_batman.properties`:

```
obj._batman.properties # => <#Batman.SimpleHash>
```

Interestingly, properties are added to `_batman.properties` _as needed_, so when you lookup a value on a `Batman.Object`, it adds an entry to `_batman.properties` (unless there's already one there for that key).

For example, a `Batman.Object` starts with _no_ properties at all:

```coffeescript
obj = new Batman.Object
obj._batman.properties # => undefined
```

When you lookup with `get`, the `properties` hash is created:

```coffeescript
obj.get('someProp')
obj._batman.properties # => <#Batman.SimpleHash>
obj._batman.properties.length # => 1
```

When you lookup another key, another entry is added to `properties`:

```coffeescript
obj._batman.properties.length # => 1
obj.get('someOtherProperty')
obj._batman.properties.length # => 2
```

#### Accessor

Every `Batman.Property` has an accessor object. The accessor object defines `get` and `set` functions which are used for mutating the property's value. You can get inspect it with `Batman.Property::accessor`:

```
obj = new Batman.Object
obj.get('someProperty')
prop = obj.property('someProperty')
prop.accessor() # => {get: [Function], set: [Function]}
```

`accessor` is a function because batman.js actually performs a lookup. It searches:

- The object's `_batman.keyAccessors` (which is where `@accessor` puts getters and setters)
- The object's ancestor's `_batman.keyAccessors`
- The object's default accessor

The result of this lookup is cached and never reevaluated.

#### Batman.Keypath

`Batman.Keypath` is a class which extends `Batman.Property`. In short, it brings support "nested lookups". In fact, most objects use `Batman.Keypath` instances to back their properties.

## Custom Accessors
\label{sec:custom_accessors}

To define a custom accessor, you can use the normal accessor syntax to define `get`, `set`, and `unset` operations:

```coffeescript
@accessor 'myProperty',
  get: (key) -> alert "you're looking for #{key} on #{@}!"
  set: (key, value) -> alert "you want set #{key} as #{value} on #{@}!"
  unset: (key) -> alert "you want to unset #{key} on #{@}!"
```

Or the shorthand syntax to define a `get` operation only:

```coffeescript
@accessor 'myShorthandProperty', (key) -> console.log "#{key} on #{@} can't be `set` or `unset`"
```

### Storing Values
\label{sec:storing_values}

When defining custom accessors, you'll need somewhere to store the value passed to `set`. Two common patterns are to use a __JavaScript property__ or to use a __`Batman.Hash`__.

#### JavaScript Property
\label{sec:javascript_property}

The easiest way to store a value is to hide it on the object itself. Just make `get`, `set`, and `unset` all use the same "private" property of the object:

```coffeescript
@accessor 'yearsOld', ->
  get: -> @_yearsOld
  set: (key, value) -> @_yearsOld = value
  unset: -> # `unset` operations should return their previous values
    prev = @_yearsOld
    @_yearsOld = undefined
    prev
```

Batman.js handles all the observability concerns when you call `get`/`set`, so it's OK to use a plain JavaScript property inside `@accessor` functions.

#### `Batman.Hash`
\label{sec:batman_hash}

Storing values in a `Batman.Hash` provides a few benefits:

- doesn't clutter the object, no risk of name conflicts
- `Batman.Hash` is enumerable
- `Batman.Hash` is serializable

If you plan on getting properties back as JSON or iterating over them, `Batman.Hash` is a great choce. In fact, that's how `Batman.Model` does it! `record.get('attributes')` returns the `Batman.Hash` where values are stored.

You can do it like this:

```coffeescript
class Tuxedo extends Batman.Object
  @accessor 'pieces', -> new Batman.Hash
  @delegate 'bowTie', 'jacket', 'vest', to: 'pieces'
```

Now, calling `tuxedo.set('bowTie', 'plaid')` will actually set `tuxedo.pieces.bowTie`. Likewise, `tuxedo.get('bowTie')` will actually get `tuxedo.pieces.bowtie`.

`Batman.Model` uses a `Batman.Hash` (at `attributes`) to store its values.

### Computed Properties
\label{sec:computed_properties}

The simplest use case of a custom accessor is a computed property:

```coffeescript
class Person extends Batman.Object
  @accessor 'isLikeable', ->
    @get('isCourteous') and @get('hasSenseOfHumor')
```

Defining convenience accessors like this makes view bindings more concise and makes your code more maintainable.

### Default Values
\label{sec:default_values}

Custom accessors allow you to define sensible defaults for your application:

```coffeescript
@accessor `pointsScored`,
  get: -> @_pointsScored || 0
  set: (k, v) -> @_pointScored = v
```

### Promise Accessors
\label{sec:promise_accessors}

Accessors which must be resolved asynchronously can be defined with the `promise:` option:

```coffeescript
@accessor 'remoteProperty',
  promise: (deliver, key) ->
    new Batman.Request
      url: "/api/endpoint"
      success: (data) ->
        deliver(null, data)
      error: (err) ->
        deliver(err, null)
    undefined
```

The function takes two arguments:

- `deliver` is a function which should be called when the asynchronous action is finished. It should be called with `deliver(err, newValue)`
- `key` is the key passed to `get`.


In this case, the property will behave like this:

```coffeescript
object.get('remoteProperty') # => undefined
# When the XHR request is successful, 'remoteProperty' will be set:
object.get('remoteProperty') # => "some value"
```

__Notes about promise accessors__

- View bindings will automatically be updated when `"remoteProperty"` is updated by the deliver function. the `newValue` is assigned with `set`, so all dependencies and observers are notified.

- _If the promise function returns a value_ other than `undefined`, it will be treated as an early return and cached by batman.js. Make sure your promise functions return `undefined`!

- Promise accessors _don't track their sources_. It's not possible because of their asynchronous nature: batman.js can't use its global source tracker.

### Pseudo-Promise Accessors
\label{sec:pseudo_promise_accessors}

Sometimes, it's better to return an incomplete-but-useful object.

```coffeescript
class Hero extends Batman.Object
  @accessor 'sidekick', ->
    sidekick = new Sidekick
    sidekick.load(url: "/heroes/#{@get('id')}/sidekick")
    sidekick
```

Since the `sidekick` is returned, it can be put into a view right away. When it's async load finishes, its attributes will be populated and the view will be updated.

### Respond to Input Values
\label{sec:respond_to_input_values}

In your `set` operation, you can define logic to handle the values which are passed into the property. For example, imagine a text input which should _only_ accept digits and  automatically dial the number when it is 10 digits long:

```html
<input type='text' data-bind='phoneNumber'></input>
```

It could be implemented this way[^observable_would_work]:

```coffeescript
@accessor 'phoneNumber',
  get: -> @_phoneNumber
  set: (k, v) ->
    if /^\d+$/.exec(v)
      @_phoneNumber = "#{v}"
    if @_phoneNumber.length == 10
      phoneCall = new PhoneCall(number: @_phoneNumber)
      phoneCall.dial()
    @_phoneNumber
```

You can even prevent `set`ting altogether:

```coffeescript
@accessor 'protectedProperty',
  get: (key) -> @_protectedProperty
  set: (key, value) ->
    if @get('isFrozen')
      return undefined
    else
      @_protectedProperty = value
```

`Batman.Model` uses this approach to prevent `set`ting values while a record is undergoing a storage operation.

## Extract Method: Accessors as Tiny Bits of Logic
\label{sec:extract_method}

It's better to have several, simple accessors than one giant accessor for two reasons:

- Small accessors are easier to maintain
- Batman.js will perform fewer operations when one of its sources changes.

Consider `Sandwich`:

```coffeescript
class Sandwich extends Batman.Object
  @accessor 'isVegan', ->
    !@get('bacon') and !@get('turkey') and !@get('tunaSalad') and
    !@get('cheddar') and !@get('provolone') and
    !@get('mayonnaise')

  @accessor 'isPescatarian', ->
    !@get('bacon') and !@get('turkey')
```

### Maintainability
\label{sec:maintainability}

There's some logic to be extracted to its own accessor, `hasMeat`:

```coffeescript
class Sandwich extends Batman.Object
  @accessor 'hasMeat', ->
    !@get('bacon') and !@get('turkey')

  @accessor 'isVegan', ->
    !@get('hasMeat') and !@get('tunaSalad') and
    !@get('cheddar') and !@get('provolone') and
    !@get('mayonnaise')

  @accessor 'isPescatarian', ->
    !@get('hasMeat')
```

In this case, _less code_ means:

- easier to test, since you can test `hasMeat` independently of other accessors
- easier to maintain, since adding and removing meats is easy to do (or changing the implementation of `hasMeat`)

### Performance
\label{sec:performance}

To show how more, smaller accessors can improve performance, lets add another accessor, `hasCheese`:

```coffeescript
class Sandwich extends Batman.Object
  @accessor 'hasMeat', ->
    !@get('bacon') and !@get('turkey')

  @accessor 'hasCheese', ->
    !@get('cheddar') and !@get('provolone')

  @accessor 'isVegan', ->
    !@get('hasMeat') and !@get('tunaSalad') and
    !@get('hasCheese') and
    !@get('mayonnaise')

  @accessor 'isPescatarian', ->
    !@get('hasMeat')
```

Given these two implementations, let's see what happens when we add mayonnaise to an otherwise-vegan sandwich:

```coffeescript
tomatoSandwich = new Sandwich(tomato: true)
tomatoSandwich.get('isVegan') # => true
tomatoSandwich.set('mayonnaise', true)
```

`tomatoSandwich.isVegan` is aware of its sources, and when one source changes, it recaluclates itself. Here's the `get` stack triggered by this operation for both implementations:

| First implementation | Second implementation |
|------|-----|
|`bacon`| `hasMeat`|
|`turkey`| `hasCheese`|
|`cheddar`| `tunaSalad`|
|`provolone`| `mayonnaise`|
|`tunaSalad`| |
|`mayonnaise`| |

In the second implementation, batman.js executed fewer property lookups which amounts to a performance improvement.

However, there are cases where this won't provide an improvement. Adding `provolone` results in the same number of lookups:

```coffeescript
tomatoSandwich = new Sandwich(tomato: true)
tomatoSandwich.get('isVegan') # => true
tomatoSandwich.set('provolone', true)
```
| First implementation | Second implementation |
|------|-----|
|`bacon`| `hasMeat`|
|`turkey`| `hasCheese`|
|`cheddar`| `cheddar`|
|`provolone`| `provolone`|


And adding `bacon` is actually more efficient in the first implementation:

```coffeescript
tomatoSandwich = new Sandwich(tomato: true)
tomatoSandwich.get('isVegan') # => true
tomatoSandwich.set('bacon', true)
```

| First implementation | Second implementation |
|------|-----|
|`bacon`| `hasMeat`|
| | `bacon`|

## Default Accessor as Batman.js's `method_missing`
\label{sec:default_accessor}

In Ruby[^and_others], `method_missing` provides a tremendous opportunity to design efficient, human-friendly APIs. In batman.js, defining the _default accessor_ is analogous to defining `method_missing`.

### All Batman.Objects have a default accessor
\label{sec:all_objects_default_accessor}

You can `get` and `set` properties on `Batman.Object` even when those properties weren't explicitly defined with `@accessor`. This is because all `Batman.Object`s have a _default accessor_: an accessor invoked for keys which don't have specific accessors.

### Redefining the Default Accessor
\label{sec:redefining_the_default_accessor}

You can redefine the default accessor for a `Batman.Object` subclass by calling `@accessor` with no keys:

```coffeescript
class NoPropertiesObject extends Batman.Object
  @accessor
    get: (key) -> alert "you're looking for #{key} on #{@}!"
    set: (key, value) -> alert "you want set #{key} as #{value} on #{@}!"
    unset: (key) -> alert "you want to unset #{key} on #{@}!"
```

For example, `BaseballTeam`'s  default accessor accepts players for positions, but rejects invalid positions. It stores the `position: player` pairs in the `members` hash.

```coffeescript
class BaseballTeam extends Batman.Object
  POSITIONS: ["firstBase", "secondBase", "shortStop", "thirdBase",
    "catcher", "rightField", "centerField", "leftField", "pitcher"]

  @accessor 'members', -> new Batman.Hash()
  @accessor ->
    get: (key) ->
    set: (key, value) ->
      if key in @POSITIONS
        @get('members').set(key, value)
      else
        throw "#{key} isn't a valid position!"
  toJSON: ->
    @get('members').toJSON()
```

Valid positions may be assigned:

```coffeescript
padres = new BaseballTeam
padres.set('thirdBase', "Chase Headley")
padres.set('rightField', "Tony Gwynn")
```

But invalid ones may not:

```coffeescript
padres.set('quarterback', "John Elway") # => Error: quarterback isn't a valid position!
```

And explicitly-defined accessors take precendence:

```coffeescript
padres.get('members') # => Batman.Hash
padres.toJSON()       # => {rightField: "Tony Gwynn", thirdBase: "Chase Headley"}
```

### Examples of Redefining the Default Accessor
\label{sec:default_accessor_examples}

Here are a few ways that batman.js components use the default accessor:

- `Batman.Proxy` and `Batman.SetProxy` pass messages on to their `target`s.
- `Batman.Hash` and `Batman.Model` store key-value pairs so they can be serialized and iterated over (without interference from with other properties).
- `Batman.SetIndex` looks up a set whose value matches the given key.
- `Batman.UniqueSetIndex` looks up the first record whose value matches the given key.
- `Batman.ControllerDirectory` looks up controller classes by the keys passed to it.
- `Batman.I18N.LocalesStorage` fetches locales from the server.

## Observing Properties
\label{sec:observing_properties}

You can observe properties on `Batman.Objects`. Observers are fired when the value is updated to a _new value_ with `set`.

```coffeescript
obj = new Batman.Object
obj.set('count', 1)
obj.observe 'count', (newValue, oldValue) ->
  console.log("Count was #{oldValue}, now it's #{newValue}")
obj.set('count', 2)
# => Count was 1, now it's 2
```

However, you must `forget` these observers to prevent memory leaks:

```coffeescript
obj.forget('count')
obj.set('count', 3)
# => ... nothing ...
```

There are some other niceties to help you observe:

- `observeOnce` forgets itself after executing once
- `observeAndFire` executes itself once, then starts observing

## Batman.Object as Event Emitter
\label{sec:event_emitter}

`Batman.Object`s can also fire events.

```coffeescript
party = new Batman.Object
party.on 'announce', (message) -> alert("Surprise!! #{message}")
# ...
party.fire 'announce', "It's your birthday!"
# => "Surprise!! It's your birthday!"
```

Handlers can be removed with `off`:

```coffeescript
party.off('announce')
```

There are some niceties to help you with events:

- `once` fire after the first event, then `off`s itself
- `allow`/`prevent`/`allowAndFire` enable controlling whether handlers are fired or suppressed.
- `event` returns the underlying `Batman.Event` instance

[^and_others]: Of course, many languages include this feature! However, JavaScript isn't one of them: [`__noSuchMethod__`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/noSuchMethod) isn't reliable.

[^observable_would_work]: Using `@observe('phoneNumber', @_dialIfTenDigits)` would be another fine implementation.


