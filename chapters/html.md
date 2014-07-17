# HTML, Bindings and Filters
\label{cha:batman_view_html}

## Displaying Records
\label{sec:displaying_records}

### Display All Records Sorted by Name
\label{sec:sort_records}

Use `Set::sortedBy` on the Model's `all` set:

```html
<ul>
  <li data-foreach-station="Station.all.sortedBy.name"></li>
</ul>
```

Or `sortedByDescending`:

```html
<ul>
  <li data-foreach-station="Station.all.sortedByDescending.score"></li>
</ul>
```

### Display Model Errors in a Form
\label{sec:formfor_errors}

- An element with class `errors` will automatically have errors listed inside it.
- Form fields will automatically get class `error` if their fields have errors.

```html
<form data-formfor-comment='currentComment'>
  <div class='errors'></div>
  <input type='text' data-bind='comment.message' />
</form>
```

### Dynamically Render a Partial from a Record
\label{sec:dynamic_partials}

Batman.js's `data-partial` binding takes a string, not a keypath, so it's not good for dynamically selecting a partial to render. However, `data-view` does take a keypath, so you can use that.

First, make an accessor on your model which returns a `Batman.View` class (or instance):

```coffeescript
class App.Animal extends Batman.Model
  @accessor 'toPartial', ->
    if @get('isNew')
      App.AnimalsNewView
    else
      App.AnimalsShowView
```

Then, use that accessor in a `data-view` binding:

```html
<div data-view='animal.toPartial'>
  <!-- will use either AnimalsNewView or AnimalsShowView, depending on `animal.isNew` -->
</div>
```

(See also Section~\ref{sec:named_partials} for accessing specific views )

### Access Specific Views from a Record
\label{sec:named_partials}

_Thanks to [@bradstewart](https://github.com/bradstewart) for this one!_

To have customized, named partials attached to your records, define accessors that return a `Batman.View` instance (or class):

```coffeescript
class App.Animal extends Batman.Model
  partial: (partialName) ->
    source = "animals/_#{partialName}"
    animal = @
    new App.AnimalView({source, animal})

  @accessor 'show', -> @partial('show')
  @accessor 'create', -> @partial('create')
```

Then, you could reach these views from bindings:

```html
<div data-view='animal.show'></div>
```

(See also Section~\ref{sec:dynamic_partials} for using `Model::toPartial` to render partials)

## Input Bindings
\label{sec:input_bindings}

### Bind to a Select
\label{sec:select_binding}

Use `data-bind`.

```html
<select data-bind="label.quantity">
  <option value=''>How many?</option>
  <option value='1'>1</option>
  <option value='2'>2</option>
  <option value='3'>3</option>
  <option value='4'>4</option>
  <option value='5'>5</option>
</select>
```

### Get Select Binding Options from a Collection
\label{sec:dynamic_select_binding}

Put a `data-foreach` binding on the `option` tag, then be sure to use `data-bind` to fill the tag and `data-bind-value` to set the value attribute.

```html
<select data-bind="phone.location">
  <option
    data-foreach-location="PhoneNumber.LOCATIONS"
    data-bind='location'
    data-bind-value='location.id'
    >
  </option>
</select>
```

This works with JavaScript arrays and `Batman.Set`s. To include a blank value, add an `<option>` before the `data-foreach`:

```html
<select data-bind='newCountryId'>
  <option value="">None</option>
  <option
    data-foreach-country='availableCountries'
    data-bind='country.name'
    data-bind-value='country.id'
    >
  </option>
</select>
```

### Bind to Radio Buttons for Multiple-Choice Properties
\label{sec:radio_buttons}

To use radio buttons for a multiple choice property, use `data-bind` and assign the value of each button:

```html
  <input type='radio' name='kind' data-bind='location.kind' value='Directory' />
  <input type='radio' name='kind' data-bind='location.kind' value='Entry' />
```

You can use `data-addclass` to style the selected button:

```html
<label data-addclass-selected="country.government | equals 'Autocracy'">
  <input type='radio' name='government' value='Autocracy' data-bind='country.government' />
  Monarchy
</label>

<label data-addclass-selected="country.government | equals 'Democracy'">
  <input type='radio' name='government' value='Democracy' data-bind='country.government' />
  Democracy
</label>

<label data-addclass-selected="country.government | equals 'Oligarchy'">
  <input type='radio' name='government' value='Oligarchy' data-bind='country.government' />
  Oligarchy
</label>
```

## Other Bindings & Filters
\label{sec:other}

### Bind to Specific HTML Attributes
\label{sec:bind_to_html_attributes}

Use `data-bind-#{attr}` to bind to _any_ attribute of a node. For example, to bind the `src` attribute of an `<img />`:

```html
<img data-bind-src='country.flag_url' />
```

To bind to the `href` of an `<a/>`:

```html
<a href='country.government_website_url' target="_blank">See Official Homepage</a>
```

### Fire Controller Action without Changing the URL
\label{sec:fire_controller_action}

To execute a controller action _without_ any routing, use `dispatch`, passing it an action name and any arguments for that action:

```html
<a data-event-click='controllers.posts.dispatch | withArguments "new", blogPost'>
  Comment on this Post
</a>
```

This will fire `posts#new`, passing it `blogPost`, without changing the current URL.

### Add a Custom Filter
\label{sec:custom_filter}

`Batman.Filters` is a JavaScript objects whose properties are view filters. Add a filter by assigning a new key:

```coffeescript
Batman.Filters.removeDigits = (string) ->
  string?.replace(/[0-9]/g, '')
```
Now you can use it in view bindings:

```html
<span data-bind='myString | removeDigits'></span>
```

Bindings _must_ be prepared to handle `undefined` values. They're often undefined during initialization and tear-down of views.

### Use Moment.js for Date/Time Formats
\label{sec:moment}

```coffeescript
Batman.Filters.moment = (date, format) ->
    moment(date).format(format)
```

```html
<span data-bind='currentEvent | moment "MMMM Do YYYY, h:mm:ss a"'></span>
```


### Send Views or Clicks to Google Analytics
\label{sec:view_or_click_tracking}

_Thanks to the [Shopify team](https://github.com/batmanjs/batman/pull/835) for this one!_

Implement `App.EventTracker` and use `data-track` bindings.

Your app should have an `EventTracker` class that responds to `track(eventName, keypathValue, node)`. For example:

```coffeescript
class App.EventTracker
  track: (type, item, node) ->
    if type is 'click'
      _gaq?.push(['_trackEvent', item.get('category'), item.get('label')])
    else if type is 'view'
      _gaq?.push(['_trackPageview', window.location.href])
```

_(`view` is triggered when the page is loaded)_

Then, put `data-track` bindings in your HTML. For example, to track that an `item` was clicked:

```html
<ul>
  <li data-foreach-item="items">
    <a data-route='routes.items[item]' data-track-click='item' data-bind='item.label'></a>
  </li>
</ul>
```