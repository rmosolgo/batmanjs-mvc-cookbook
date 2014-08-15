# Views
\label{cha:batman_view}

## Custom Views
\label{sec:custom_views}

### Initialize a jQuery Component
\label{sec:jquery_initialization}

The easiest way to use jQuery components is to define custom views. These views can be used whenever you need that component. For example, here is a
custom view for the jQuery UI datepicker:

```coffeescript
class App.jQueryDatePickerView extends Batman.View
  viewDidAppear: ->
    if !@_initialized
      $(@node).datepicker
        changeMonth: true
        changeYear: true
      @_initialized = true
```

`viewDidAppear` is part of the view lifecycle API. Note that it may be called more than once, since it "bubbles up" from subviews.

When you need the component, use a `data-view` binding:

```html
<input type='text' data-bind='person.birthdate' data-view='jQueryDatePickerView'/>
```

### Transition Between Pages
\label{sec:view_transitions}

Use the view lifecycle API and implement default views (Section~\ref{sec:default_views}) to provide transitions between pages.

Here's an implementation with jQuery fade:

```coffeescript
class App.FadeView extends Batman.View
  viewWillAppear: ->
    $(@get('node')).hide()

  viewDidAppear: ->
    $(@get('node')).fadeIn('fast')

  viewWillDisappear: ->
    $(@get('node')).fadeOut('fast')
```

Now, implement default views to inherit from `App.FadeView`:

```coffeescript
class App.CountriesShowView extends App.FadeView
class App.CountriesEditView extends App.FadeView
```

If you add to any of the lifecycle hooks above, be sure to call `super`:

```coffeescript
class App.CountriesIndexView extends App.FadeView
  viewDidAppear: ->
    super
    @_initialize() if !@_initialized
```

### Cascading Fade with Iteration View
\label{sec:cascade_fade_iteration_view}

If we combine the power of Batman's view inheritance with the view lifecycle API, we can accomplish a cascading fade:

```coffeescript
class App.CascadingFadeIterationView extends Batman.IterationView
  viewWillAppear: ->
    $(@get('node')).hide()

  viewDidAppear: ->
    node = $(@get('node'))
    @superview._counter ||= 0
    setTimeout (->
      node.fadeIn(200)
    ), @superview._counter += 200
```

```html
<div data-foreach-country="countries" data-view="CascadingFadeIterationView">
  <!-- will fade in incrementally -->
</div>
```

### Define Custom HTML
\label{sec:custom_html}

Use the `source` property in a view definition:

```coffeescript
class App.CountriesFlashView extends Batman.View
  source: 'countries/flash'
```

Then, when the view is rendered, it will get its HTML from that path:

```html
<div data-class='CountriesFlashView'>
  <!-- will be populated with countries/flash -->
</div>
```

\begin{aside}
\label{aside:default_view_sources}
\heading{Default View Sources}

\noindent When a view is rendered implicitly, you don't have to define its `source`.

Batman.js will provide a default source composed of the controller's `routingKey` and the action name:

```coffeescript
"#{routingKey}/#{actionName}"
```

This is similar to controller's default view lookup (see Section~\ref{sec:default_views}).

\end{aside}

### Provide HTML for Multiple Yields
\label{sec:content_for_multiple_yields}

A single view can provide HTML for multiple yields. In the source HTML, use a `data-contentfor` binding. Everything inside that element will be rendered inside the named yield _instead of_ the main yield.

```html
<!-- rendered inside data-yield='header': -->
<div data-contentfor="header">
  <h1>
    Editing
    <span data-bind='product.name'>
  </h1>
</div>

<!-- rendered inside data-yield='main': -->
<form data-formfor-p='product' data-event-submit='saveProduct'>
 <!-- ... -->
</form>
```

(This is analagous to [Rails' `content_for`](http://api.rubyonrails.org/classes/ActionView/Helpers/CaptureHelper.html#method-i-content_for) view helper.)

## User Input
\label{sec:user_input}

### Deactivate When Save is Pending
\label{sec:deactivating_form}

- Use accessor `savingMessage` as an indicator that the save operation is pending.
- Use `data-bind-disabled` to disable inputs while the save operation is pending.
- Show `savingMessage` on the submit button if it's present, otherwise, show the default message.
- Don't forget to unset `savingMessage` when the operation is finished!

Here is an example. In the HTML, note the `data-bind-disabled` checking for `savingMessage` and the default value provided to the submit input.

```html
<form
  data-formfor-country='currentCountry'
  data-event-submit='save | withArguments country'
  >
  <input data-bind='country.name' data-bind-disabled='savingMessage'/>
  <input type='submit' data-bind-value='savingMessage | default "Save Changes"'>
</form>
```

In the view, note the usage of `_deactivate` and `_activate`:

```coffeescript
class App.CountryEditView extends Batman.View
  save: (country) ->
    @_deactivate()
    country.save (err, record) =>
      @_activate()
      if err? and !(err instanceof Batman.ErrorsSet)
        throw err

  _deactivate: ->
    @set('savingMessage', 'Saving...')

  _activate: ->
    @unset('savingMessage')
```

### Sterilize Input Values
\label{sec:sterilize_inputs}

Define custom accessors with `get` and `set` functions, then bind to them in your HTML. In this case, `numberFormatted` is a proxy to `number`, removing non-digit characters before setting.

```coffeescript
class App.PhoneNumberEditView extends Batman.View
  @accessor 'numberFormatted',
    get: -> @get('number')
    set: (key, value) ->
      @set('number', value.replace(/[^0-9]/g, ''))
```

Then, bind to the proxy accessor rather than the underlying one:

```html
<input type='text' data-bind='numberFormatted'></input>
```

## Managing Contexts
\label{sec:context}

### Observe Objects Within the View's Scope
\label{sec:observe_in_view}

To observe items in the view, set up observers on the view instance. It's better to define observers on the _view_ rather than the _model_, because that way, it will be properly cleaned up when the view is finished.

```coffeescript
class App.CountriesEditView extends Batman.View
  viewDidAppear: ->
    return if @_observing
    @observe 'controller.currentCountry.government', (newValue, oldValue) ->
      if oldValue is "Anarchy"
        alert "Let there be order!"
      else if newValue is "Anarchy"
        alert "Let there be chaos!"
    @_observing = true
```

`viewDidAppear` is part of the view lifecycle API. Note that it may be called more than once, since it "bubbles up" from subviews.

### Lookup Values in the View's Context
\label{sec:lookup_in_context}

You may have noticed that bindings can "see" values that can't be retrieved with `get`. This is because bindings use `lookupKeypath`, which climbs the view hierarchy, finally landing in the global scope. You can use `lookupKeypath` in your views, too.

For example, if `currentCountry` is set on the controller and `currentUser` is set on the current App, you could access them like this:

```coffeescript
class App.CountriesEditView extends Batman.View
  save: ->
    country = @lookupKeypath('currentCountry')
    user = @lookupKeypath('currentUser')
    country.set('edited_by', user)
    country.save()
```
