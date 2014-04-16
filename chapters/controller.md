# Controller
\label{cha:batman_controller}

## Render a View _after_ the Data Has Loaded
\label{sec:defer_render}

Use `@render(false)` to disable the implicit, inline render. Then, call `@render()` in the operation's call back. Be sure to use a fat arrow (`=>`) when passing the callback so that `@` refers to the controller.

```coffeescript
class App.PostsController extends App.ApplicationController
  show: (params) ->
    App.Posts.find params.id, (err, record) =>
      throw err if err?
      @set('currentPost', record)
      @render()

    @render(false) # prevent implicit render
```

## Editing & Creating Records with Reusable Forms
\label{sec:reusable_forms}

To reuse forms for `edit` and `new` actions, make both actions assign records to the same accessor. For example, use `currentPost`:

```coffeescript
class App.PostsController extends App.ApplicationController
  new: ->
    @set 'currentPost', new App.Post

  edit: (params) ->
    App.Post.find params.id, (err, record) =>
      @set 'currentPost', record
```

Then, create form HTML and save it as `posts/_form`:

```html
<form data-formfor-post='currentPost' data-event-submit='save | withArguments currentPost'>
  <input type='text' data-bind='post.name' />
  <textarea data-bind='post.content'></textarea>
  <input type='submit' value='Save Post' />
</form>
```

Now, your HTML for `posts/edit` and `posts/new` can both render the same form:

```html
<div data-partial='posts/_form'></div>
```

## Wiping a Record's Changes If They Aren't Saved

Watch [Model::transaction](https://github.com/batmanjs/batman/pull/992) for integration into batman.js master.

Use `Model::transaction` to create deep, ephemeral copies of records:

```coffeescript
class App.PostsController extends App.ApplicationController
  edit: (params) ->
    App.Post.find params.id, (err, record) =>
      postTransaction = record.transaction()
      @set 'currentPost', postTransaction
```

Transactions respond to `save` by applying changes to the base record, then saving it. So, you can treat the transaction just like a normal record: call `save` to save changes.

If the transaction is never saved, its changes are forgotten and never affect the base record.

## Saving and Destroying Records
\label{sec:saving}

The best way to save a record is to define the function on the controller (or view[^or_view]) and invoke it from a `data-event-submit` binding.

Here's some example HTML:

```html
<form
  data-formfor-country='currentCountry'
  data-event-submit='save | withArguments country'
  >
  <input type='text' data-bind='country.name' />
  <input type='submit' value='Save' />
</form>
```

Then, implement save on the controller:

```coffeescript
class App.CountriesController extends App.ApplicationController
  save: (country) ->
    country.save (err, record) ->
      if err? and !(err instanceof Batman.ErrorsSet)
        throw err
      else
        Batman.redirect('/countries')
```

For destroying, implement a function on the controller:

```coffeescript
  destroy: (country) ->
    if confirm("Are you sure you want to delete #{country.get('name')}?")
      country.destroy (err, record) ->
        throw err if err?
        Batman.redirect("/countries")
```

Then use a `data-event` to trigger it:

```html
<a data-event-click='destroy | withArguments country'>
  Destroy
  <span data-bind='country.name'></span>
</a>
```

## Render Views into a Modal

See a [live, annotated example](http://bl.ocks.org/rmosolgo/10606657) of this implementation ([source](https://gist.github.com/rmosolgo/10606657)).

Set up a `data-yield='modal'`:

```html
<div id='modal'>
  <div data-yield='modal'></div>
</div>
```

Observe `Batman.DOM.Yield.get('yields.modal.contentView')` for when a view is rendered. Open the modal inside that observer:

```coffeescript
class @App extends Batman.App
  @on 'run', ->
    Batman.DOM.Yield.observe 'yields.modal.contentView', (newValue, oldValue) ->
      if newValue?
        $('#modal').modal('show')
```

_(See Aside~\ref{aside:modals} for another way to open the dialog.)_

Create a function on ApplicationController to close the dialog and `die` the view and add it as a `beforeAction`:

```coffeescript
class App.ApplicationController extends Batman.Controller
  closeDialog: ->
    $('.modal').modal('hide')
    modalYield = Batman.DOM.Yield.get('yields.modal')
    modalYield.get('contentView')?.die()
    modalYield.set('contentView', undefined)

  @beforeAction @::closeDialog
```

For actions which should render into the modal, explicitly render: `@render(into: 'modal')`. The modal will be opened automatically by the observer:

```coffeescript
class App.AnimalsController extends App.ApplicationController
  # ...
  edit: (params) ->
    App.Animal.find params.id (err, animal) ->
      @set 'currentAnimal', animal
    @render(into: 'modal')
```

Since you're just calling `@render`, it will still lookup a view class to intantiate (see Section~\ref{sec:default_views}).

See also Section~\ref{sec:fire_controller_action} to open a model without changing the URL.

\begin{aside}
\label{aside:modals}
\heading{Another Way to Open a Modal}

\noindent `@render` returns the rendered view, so that offers another way to open the modal.

Listen for the view's `viewDidAppear` event:

```coffeescript
class App.AnimalsController extends App.ApplicationController
  # ...
  edit: (params) ->
    App.Animal.find params.id (err, animal) ->
      @set 'currentAnimal', animal

    view = @render(into: 'modal')
    view.on 'viewDidAppear', ->
      $('#modal').modal('show')
```

You could move this functionality into a function on `ApplicationController`:

```coffeescript
class App.ApplicationController extends Batman.Controller
  dialog: (options={}) ->
    options = Batman.mixin({}, {into: "modal"}, options)
    @render(options).on 'viewDidAppear', ->
      $('#modal').modal('show')
```

Then use that function to render actions into dialogs:

```coffeescript
class App.AnimalsController extends App.ApplicationController
  # ...
  edit: (params) ->
    App.Animal.find params.id (err, animal) ->
      @set 'currentAnimal', animal
    @dialog('modal')
```

\end{aside}


## Controller Actions Have Default Views
\label{sec:default_views}

When a controller action renders, it looks for a view class. If it doesn't find one, it generates it on the fly. You can implement these default view classes to extend default functionality. Here's how the default view classes are determined:


Controller Name | Action | View Name
=== | === | ===
App.CountriesController | show | App.CountriesShowView
App.CountriesController | new | App.CountriesNewView
App.CountriesController | index| App.CountriesIndexView
App.CountriesController | edit | App.CountriesEditView

If you define those view classes, they will be instantiated when the controller actions render. For example,

```coffeescript
class App.CountriesController extends App.ApplicationController
  show: (params) ->
    App.Country.find params.id, (err, record) =>
      @set('currentCountry', record)
    # will render App.CountriesShowView

  new: ->
    @set 'currentCountry', new App.Country
    # will render App.CountriesNewView

  index: ->
    @set('countries', App.Country.get('all'))
    # will render App.CountriesIndexView

  edit: (params) ->
    App.Country.find params.id, (err, record) =>
      @set('currentCountry', record)
      @render() # will render App.CountriesEditView
    # defer rendering until `find` resolves
    @render(false)
```

[^or_view]: Since bindings use `View::lookupKeypath`, they start at the "bottom" of the view hierarchy and work their way up. They'll find functions on the view or controller.