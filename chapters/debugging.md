# Debugging
\label{cha:debugging}

## Looking Up Values in a View Context
\label{sec:view_context}

Batman.js ships with a global function called `$context` which returns the `Batman.View` for a given HTML node.

1. In Google Chrome, select a node with Right Click - "Inspect Element". That node is now `$0` in the JavaScript console.
2. In the JavaScript console, lookup the node's view: `view = $context($0)`
3. Use `View::lookupKeypath` to get values in the view's context: `view.lookupKeypath('company.name')`

## Calling the Debugger During Binding Initialization
\label{sec:data_debug}

Batman.js development builds ship with a `data-debug` binding which calls `debugger` when it is intialized:

```html
<span data-debug='true' />
```

## Inspecting Models in Memory
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

