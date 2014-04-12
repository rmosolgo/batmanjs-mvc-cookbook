# Filters
\label{cha:batman_filters}

## Add a Custom Filter
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

## Use Moment.js for date/time formats
\label{sec:moment}

```coffeescript
Batman.Filters.moment = (date, format) ->
    moment(date).format(format)
```

```html
<span data-bind='currentEvent | moment "MMMM Do YYYY, h:mm:ss a"'></span>
```