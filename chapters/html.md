# HTML

## Display all records sorted by name
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

## Bind to a Select
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

## Get Select Binding Options From a Collection
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

## Display Model Errors in `data-formfor`
\label{sec:formfor_errors}

- An element with class `errors` will automatically have errors listed inside it.
- Form fields will automatically get class `error` if their fields have errors.

```html
<form data-formfor-comment='currentComment'>
  <div class='errors'></div>
  <input type='text' data-bind='comment.message'></input>
</form>
```

## Bind to a Radio Buttons for Multiple-Choice Properties
\label{sec:radio_buttons}

To use radio buttons for a multiple choice property, use `data-bind` and assign the value of each button:

```html
  <input type='radio' data-bind='location.kind' value='Directory'></input>
  <input type='radio' data-bind='location.kind' value='Entry'></input>
```

You can use `data-addclass` to style the selected button:

```html
<label data-addclass-selected="country.government | equals 'Autocracy'">
  <input type='radio' name='government' value='Autocracy' data-bind='country.government'></input>
  Monarchy
</label>

<label data-addclass-selected="country.government | equals 'Democracy'">
  <input type='radio' name='government' value='Democracy' data-bind='country.government'></input>
  Democracy
</label>

<label data-addclass-selected="country.government | equals 'Oligarchy'">
  <input type='radio' name='government' value='Oligarchy' data-bind='country.government'></input>
  Oligarchy
</label>
```

## Fire Controller Action without Routing to It
\label{sec:controller_action}

Use `executeAction`, passing it an action name and any arguments for that action:

```html
<a data-event-click='controllers.blogPostComments.executeAction | withArguments "new", blogPost'>
  Comment on this Post
</a>
```