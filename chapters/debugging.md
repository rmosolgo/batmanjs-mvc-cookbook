# Debugging
\label{cha:debugging}

## Live Debugging
\label{sec:live_debugging}

### Looking Up Values in a View Context
\label{sec:view_context}

Batman.js ships with a global function called `$context` which returns the `Batman.View` for a given HTML node.

1. In Google Chrome, select a node with Right Click - "Inspect Element". That node is now `$0` in the JavaScript console.
2. In the JavaScript console, lookup the node's view: `view = $context($0)`
3. Use `View::lookupKeypath` to get values in the view's context: `view.lookupKeypath('company.name')`

### Calling the Debugger During Binding Initialization
\label{sec:data_debug}

Batman.js development builds ship with a `data-debug` binding which calls `debugger` when it is intialized:

```html
<span data-debug='true' />
```

### Inspecting Models in Memory
\label{sec:models_in_memory}

You can always see all the loaded records in the model's `loaded` set:

```coffeescript
allProducts = App.Product.get('loaded')
allProducts.get('length') # => Total number of loaded products
allProducts.get('first')  # => Example loaded product
```

You can find a specific record by using an index on the `loaded` set:

```coffeescript
product22 = App.Product.get('loaded.indexedBy.id').get(22)
product22.get('id') # => 22
```

## Common Errors
\label{sec:common_errors}

### "Can't Save While In State Error"
\label{sec:cant_save}

This error also appears as "Can't save while in state saving" or "Can't save while in state destroyed", or many other states.

A `Batman.Model` instance has a state machine that governs what persistence actions may be performed on it at any time. Before executing `save`, the record checks its lifecycle, and if the save is not allowed, it throws this error. Depending on what state is reported by the error, this could be caused by:

- `error`: Likely there was a storage error when the record was first saved, which caused the record to enter state "error"
- `saving`: `save` was likely called twice. The second `save` was fired before the first one finished
- `destroyed`: The record was already destroyed, so it can't be saved again.

This problem is tough to avoid, and tough to get out of once the record is in state "error". To work around it, consider using `Model::transaction` (currently in batman-master), which uses a deep copy of the record for editing rather than the actual record. This way, the deep copy may get into state "error", but the user can always go back and start over.


### "HierarchyRequestError: Node cannot be inserted at the specified point in the hierarchy"
\label{sec:missing_html}
This happens with `batman-rails` when the JavaScript application requests HTML which isn't found by the server. Check your network logs: one of the HTML requests is returning the _layout view_, not the HTML template.

To fix, either:

- add the HTML template which is requested by the JavaScript application
- define a different `source` for the view which is requesting the HTML