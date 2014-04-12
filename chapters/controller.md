# Controller

## Render a View _after_ the data has loaded
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

## Putting Records In Reusable Forms
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

## Wiping A Record's Changes If They Aren't Saved

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


[^or_view]: Since bindings use `View::lookupKeypath`, they start at the "bottom" of the view hierarchy and work their way up. They'll find functions on the view or controller.