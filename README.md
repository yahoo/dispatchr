# Dispatchr [![Build Status](https://travis-ci.org/yahoo/dispatchr.svg?branch=master)](https://travis-ci.org/yahoo/dispatchr) [![Dependency Status](https://david-dm.org/yahoo/dispatchr.svg)](https://david-dm.org/yahoo/dispatchr) [![Coverage Status](https://coveralls.io/repos/yahoo/dispatchr/badge.png?branch=cover)](https://coveralls.io/r/yahoo/dispatchr?branch=cover)

A [Flux](http://facebook.github.io/react/docs/flux-overview.html) dispatcher for applications that run on the server and the client.

## Usage

For a more detailed example, see our [example application](https://github.com/yahoo/flux-example).

```js
var Dispatchr = require('dispatchr')(),
    ExampleStore = require('./example-store.js'),
    context = {};

Dispatchr.registerStore(ExampleStore);

var dispatcher = new Dispatchr(context);

dispatcher.dispatch('NAVIGATE', {});
// Action has been handled fully
```

## Differences from [Facebook's Flux Dispatcher](https://github.com/facebook/flux/blob/master/examples/flux-chat/js/dispatcher/Dispatcher.js)

Dispatchr's main goal is to facilitate server-side rendering of Flux applications while also working on the client-side to encourage code reuse. In order to isolates stores between requests on the server-side, we have opted to make the dispatcher and stores classes that are instantiated per request. They stores are still singletons within the scope of a request though.

In addition, action registration is done by stores as a unit rather than individual callbacks. This allows us to lazily instantiate stores as the events that they handle are dispatched. Since instantiation of stores is handled by the Dispatcher, we can keep track of the stores that were used during a request and dehydrate their state to the client when the server has completed its execution.

Lastly, we are able to enforce the Flux flow by restricting access to the dispatcher from stores. Instead of stores directly requiring a singleton dispatcher, we pass a dispatcher interface to the constructor of the stores to provide access to only the functions that should be available to it: `waitFor` and `getStore`. This prevents the stores from dispatching an entirely new action, which should only be done by action creators to enforce the unidirectional flow that is Flux.

## Dispatcher Interface

### registerStore(store)

A static method to register stores to the Dispatcher class making them available to handle actions and be accessible through `getStore` on Dispatchr instances.

### Constructor

Creates a new Dispatcher instance with the following parameters:

 * `context`: A context object that will be made available to all stores. Useful for request or session level settings.

### dispatch(actionName, payload)

Dispatches an action, in turn calling all stores that have registered to handle this action.

 * `actionName`: The name of the action to handle (should map to store action handlers)
 * `payload`: An object containing action information.

### getStore(storeName)

Retrieve a store instance by name. Allows access to stores from components or stores from other stores.

### waitFor(stores, callback)

Waits for another store's handler to finish before calling the callback. This is useful from within stores if they need to wait for other stores to finish first.

  * `stores`: A string or array of strings of store names to wait for
  * `callback`: Called after all stores have fully handled the action

### dehydrate()

Returns a serializable object containing the state of the Dispatchr instance as well as all stores that have been used since instantiation. This is useful for serializing the state of the application to send it to the client.

### rehydrate(dispatcherState)

Takes an object representing the state of the Dispatchr instance (usually retrieved from dehydrate) to rehydrate the instance as well as the store instance state.

## Store Interface

Dispatchr expects that your stores use the following interface:

### Constructor

The store should have a constructor function that will be used to instantiate your store using `new Store(dispatcherInterface)` where the parameters are as follows:

  * `dispatcherInterface`: An object providing access to dispatcher's waitFor and getStore functions
  * `dispatcherInterface.getContext()`: Retrieve the context object that was passed
  * `dispatcherInterface.getStore(store)`
  * `dispatcherInterface.waitFor(store[], callback)`

```js
function ExampleStore(dispatcher) {
    this.dispatcher = dispatcher;
    this.getInitialState();
}
```

It is also recommended to extend an event emitter so that your store can emit `change` events to the components.

```js
util.inherits(ExampleStore, EventEmitter);
```


### storeName

The store should define a static property that gives the name of the store. This is used for accessing stores from the dispatcher via `dispatcher.getStore(name)`.

```js
ExampleStore.storeName = 'ExampleStore';
```

### handlers

The store should define a static property that maps action names to handler function names. These functions will be called in the event that an action has been dispatched by the Dispatchr instance.

```js
ExampleStore.handlers = {
    'NAVIGATE': 'handleNavigate'
};
```

The handler function will be passed one parameter:

  * `payload`: An object containing action information.

```js
ExampleStore.prototype.handleNavigate = function (payload) {
    this.navigating = true;
    this.emit('change'); // Component may be listening for changes to state
};
```

### getState()

The store should implement this function that will return a serializable data object that can be transferred from the server to the client and also be used by your components.

```js
ExampleStore.prototype.getState = function () {
    return {
        navigating: this.navigating
    };
};
```

### dehydrate()

The store can optionally define this function to customize the dehydration of the store. It should return a serializable data object that will be passed to the client.

### rehydrate(state)

The store can optionally define this function to customize the rehydration of the store. It should restore the store to the original state using the passed `state`.

## License

This software is free to use under the Yahoo! Inc. BSD license.
See the [LICENSE file][] for license text and copyright information.

[LICENSE file]: https://github.com/yahoo/dispatchr/blob/master/LICENSE.md
