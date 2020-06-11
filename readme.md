
# Proposal:  Unified Element Properties

Today's UI frameworks exert considerable effort attempting to reconcile or "paper over" the many differences
between Element properties and attributes.  In React, this is manifested as the abstract "props" concept which,
when applied to DOM Elements requires a [complex mapping](https://github.com/facebook/react/blob/master/packages/react-dom/src/shared/DOMProperty.js)
of abstract prop names to corresponding DOM properties or attributes.
In Preact, an "in test" (prop in node) is used to infer whether a given abstract "prop" should be applied to the
DOM as a property or an attribute. This approach handles Custom Element properties more naturally, but it comes at
the cost of some performance and determinism.  Other rendering libraries choose to expose the decision of whether to
set a given value as a property or an attribute to the developer - Vue uses prefixes (:prop="value") to explicitly
control this.

It seems a set of new DOM methods designed specifically to address this case could be impactful. These methods would
establish a consistent approach for setting properties/attributes based on the given "virtual property name".
If widely adopted, this would lead to better interoperability between rendering libraries.  It also opens the door for
easier Custom Element upgrades through the use of overridden methods, and potentially provides a starting point for
addressing the attributes → properties upgrade issue faced by today's components.

### How is this different from setting DOM properties?

Setting a DOM (reflected) property immediately triggers any side effects associated with that property's setter.
In the DOM, this can be surprisingly expensive - assigning to the src property of an `<img>` element causes it
to be loaded, even if the value assigned is identical to the current value. This is true for most DOM properties,
including all properties governing Node contents (`.data`, `.textContent`, etc). This is mostly intuitive when working
directly with the DOM's imperative API, but has resulted in all modern frameworks implementing what are effectively
caches around DOM property access in order to achieve reasonable performance. Furthermore, both reading and writing
DOM properties incurs the cost of a binding traversal, since element behaviors are not implemented in JavaScript.

Ordering of property assignment can also be tricky to get right. Similar to the `<img>` issue noted above, assigning
to the `.src` property of an Image and then immediately assigning to the `.crossOrigin` property will cancel the
already-started request and re-issue it with updated CORS constraints. Similar ordering concerns are present for many
DOM properties - setting an Node's contents via `.data` or `.textContent`, needs to happen prior to setting properties
that refer to that content like `.datalist` and `.selected`. The behavior of attributes is more manageable in this
regard, since attribute changes are always batched. Synchronous code that modifies attributes behaves the same
regardless of the order in which they are set, even across multiple elements.

## API Sketch

```webidl
interface Element {
  void setProperty(string propertyName, any value);
  any getProperty(string propertyName);
}
```

#### Example usage:

```js
const element = document.createElement('div');
element.setProperty('class', 'demo');
document.body.appendChild(element);
element.getProperty('class');  // "demo"
element.setProperty('class', 'demo');  // ignored - value is unchanged
```

# Additional Opportunities

## Referential Equality

This would also open up an interesting possibility for simplifying how the DOM is manipulated both by developers
and through popular libraries, by providing a method to apply a set of properties to an Element where properties
with referentially-equal (or semantically equal?) values are skipped:

```webidl
interface Element {
  void setProperty(string propertyName, any value);
  any getProperty(string propertyName);
  void setProperties(map<string, any>; properties);
}
```

#### Example usage:

```js
const element = document.createElement('div');
element.setProperties({
  class: 'demo',
  id: 'demo-1'
});
document.body.appendChild(element);
element.setProperties({
  class: 'demo',  // ignored - value is unchanged
  style: { backgroundColor: 'red' }
});
```

## Observable Bindings

Given the existence of a `setProperty()` method on DOM Nodes, it becomes possible to implement automatic one-way
data binding simply by allowing developers to pass an [Observable](https://github.com/surma/observables-with-streams)
as a value. When given an Observable, setProperty obtains the current value of the Observable and applies it as it
would any static value, then subscribes to the observable. Future values propagated through the observable are
reflected automatically as if setProperty had been invoked with the new value.

```js
const element = document.createElement('h1');
const title = new Observable('initial title');
element.setProperty('title', title);  // assigns h1.title to "initial title"
document.body.appendChild(element);
title.next('updated');  // assigns h1.title to "updated"
```
