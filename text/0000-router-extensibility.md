- Start Date: 2018-01-13
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Router DSL extensibility

## Summary

Allow the router DSL to be extended to increase the router map's expressiveness and utility. Aimed at addon authors and advanced app developers.

## Motivation

The router.js file contains the high-level definition of the app's routes, and is often the first place a developer looks to begin becoming familiar with a new app or to orient herself when beginning a new feature. The better the route map is at expressing the high-level contours of the app, the better it serves app developers.

This RFC proposes 2 changes to support addon authors and app developers in enhancing the expressiveness of the route map. First, a hook would be added to allow the DSL implementation to be specified or extended. Second, any route options specified in the map function would be provided to the route. These changes would work hand-in-hand, with the DSL specifying options that would be available to the route.

### Example use cases

An authentication addon could allow a developer to wrap one or more routes with `this.authenticated` to express that certain route hierarchies require authentication before entering:

```js
import Router from 'my-auth-addon/router';

Router.map(function(){
  this.route('about');
  this.route('login');
  this.authenticated(function(){
    this.route('billing');
    this.route('preferences', { path: '/prefs' });
  });
});

export default Router;
```
Alternately, an auth library could express this as route option:

```js
import Router from 'my-auth-addon/router';

Router.map(function(){
  this.route('about');
  this.route('login');
  this.route('billing', { authenticated: true });
  this.route('preferences', { path: '/prefs', authenticated: true });
});

export default Router;
```

The router map for a mobile app that uses a "navigation stack" UI could indicate that certain routes should slide up a new layer:

```js
import RouterDSL from '@ember/routing/dsl';

let Router = EmberRouter.extend({
  dslFactory: RouterDSL.extend({
    layer(routeName, opts = {}, callback=null) {
      let optionsWithLayer = Object.assign({ newLayer: true }, opts);
      return this.route(routeName, optionsWithLayer, callback);
    };
  });
});

Router.map(function(){
  this.route('agenda', function() {
      this.route('schedule-item', { '/schedule-items/:schedule_item_id' });
      this.layer('my-schedule', function() {
        this.route('schedule-item', { '/schedule-items/:schedule_item_id' });
      });
    });
  });
})
```

In both examples, a route base class or mixin could be used to interpret options provided to the route and customize its behavior accordingly.

### Background

The Ember Router DSL currently allows the expression of the following aspects of the app's routes in the function passed to `Router#map`:

  1) The route name. This is expressed via a combination of the first argument to `this.route`, the nesting of the DSL usage, and the possible presence of the option `resetNamespace: true` as part of the second argument to `this.route`.
  2) The path, including one or more dynamic segments. This is expressed via the `path` option in the second argument to `this.route`. e.g. `path: '/a/path/with/:some_id'`

The additions of engines expanded the DSL to also support `this.mount`, which allows a developer to specify where to include a routable engine in their app.

There is currently no way via public API to use a different DSL implementation or to get access to route options from the route.

## Detailed design

### Changes to the router.js microlib

In order to accommodate route options being provided to route instances, support for handler options should be added to the router microlib first. The microlib API should be updated to accept an optional chained `withOptions` method that accepts an object where the keys are strings and the values are serializable. (Serializability should be asserted for now to ensure that usage is forward-compatible with possible future performance optimizations to the route generation and recognition implementation.)

An example of the updated API would look like this:

```js
router.map(function(match) {
  match("/posts/:id").to("showPost");
  match("/posts").to("postIndex");
  match("/posts/new").to("newPost").withOptions({ authenticated: true });
});
```

### Changes to Ember

The handler lookup method (which Ember provides to the microlib) would be called with the route name (as it is today) but now also include the options as a second argument.

The EmberRouter's [`_getHandlerFunction`](https://github.com/emberjs/ember.js/blob/ce3ccad1c3e294f022e68eb24a2985cd672a4c41/packages/ember-routing/lib/system/router.js#L554-L600) method would then be updated to return a function which accepts a second parameter, `options`. If `options` is passed, the handler lookup function would call `handler.applyRouteOptions(options)` after instantiating the handler.

Ember.Route would be updated to have a default no-op implementation of `applyRouteOptions`, which addon authors or app developers could override.

Currently, EmberRouter's [`_buildDSL`](https://github.com/emberjs/ember.js/blob/0e4c342271200d3ff6908124947433d7ecd7bb46/packages/ember-routing/lib/system/router.js#L116-L133) statically references `EmberRouterDSL`. Instead, it should reference a new property `dslFactory`, to get the class to be used for the DSL. In the case that complex logic is needed to determine this class (this is not expected to be common), a getter could be supplied. Adding this public property and documenting it would empower developers pursuing advanced use cases to subclass Ember's DSL class, or to provide their own implementation.

This implies that the contract of the DSL class would also need to be stable and documented. Let's look at the public methods on DSL:

`route(name: string, options: any, callback: ()=>any): void;`

This is the main method used in function passed to the `Router.map`. It is currently documented as part of `Router.map`'s docs [here](https://github.com/emberjs/ember.js/blob/0e4c342271200d3ff6908124947433d7ecd7bb46/packages/ember-routing/lib/system/router.js#L1335-L1375). Currently, the supported `options` are `path` and `resetNamespace`, with any other options being ignored. `path` and `resetNamespace` would continue to have the effect they have today, and in addition, they would be passed along with other options to the `applyRouteOptions` method of the route (see below).

The DSL implementation should assert that passed options are serializable in order to be forward-compatible with possible future performance optimizations to the route generation and recognition implementation.

`generate(): (match: any) => void;`

This is called by the router and is expected to return a function compatible with the router microlib's `map` method. See the [tildeio/router.js README](https://github.com/tildeio/router.js/) for details.

`mount(_name: string, options: any): void;`

This method is used to include a routable engine in an application. Supported `options` are `as`, `path`, and `resetNamespace`.

There is one additional method on the DSL implementation whose appearance suggests it might be part of the public API: `push`. In fact, it is only invoked by the DSL itself and should be made private by being renamed to `_push`.

Internally, the DSL implementation constructs another instance of the DSL class (i.e. `new DSL()`) when processing callbacks provided to `this.route`). This would be problematic if a developer subclasses the DSL class, so the internals should be changed so that each place we call `new DSL` [here](https://github.com/emberjs/ember.js/blob/master/packages/ember-routing/lib/system/dsl.js), we instead do:

```js
let DSL = this.constructor;
new DSL()
```

## How we teach this

It is not expected that beginner-level Ember developers would need to use these hooks. There is nothing made possible by this feature that can't be accomplished without it, albeit more verbosely and spread out across route definitions.

As a result, this feature would merit only passing mention in the guides and rely on API docs as the primary teaching mechanism to addon authors or advanced developers.

Addons which leverage these hooks will bear the "teaching burden" of how a developer needs to modify their `router.js` and routes file to work with the addon.

## Drawbacks

Allowing additional options passed to `this.route` to have meaning implies that any future options used by Ember core run the risk of name collision with application or addon code.

Allowing alternate DSL implementations makes future changes to the DSL API carry more risk.

In general, more public API surface area means more complexity, so the win of providing flexibility to advanced users via a lower-level API might be outweighed by the API surface area increase.

Historically, `Ember.Route` instances were supposed to be "stateless". This makes them somewhat more stateful. Is that a problem, or are we comfortable accepting that stateless routes is no longer a design goal?

## Alternatives

This risk of option name collisions could be eliminated by requiring that userland options be nested under hash named `routeOptions` or similar. For example:

```js
this.route('preferences', { path: '/prefs', routeOptions: { authenticated: true } });
```

However, this would negatively affects the readability and  expressiveness of the DSL, which is significant part of the goal of this RFC.

Instead of calling `Route#applyRouteOptions`, the router could simply call `Route#setProperties` with the DSL-provided options. This would make the availability of those options to the route to be less "opt-in" but more discoverable via debugging tools. The `applyRouteOptions` path is recommended by this RFC because it reduces the risk of route options overwriting route properties.

If it was important to avoid changes to the router.js microLib, this functionality could be provided wholly in Ember by making the DSL instance implement a `getOptions()` method, which would return a object with keys being routeName strings, and values being the options objects for those routes as they were passed to the DSL. This method would be called by Ember.Router and the results would be used to pass to the `handler.applyRouteOptions` method when each route is instantiated. It seems more elegant to bake this into the microlib, though.

### Prior art in the Ember ecosystem

An attempts at an alternative DSL implementation was made here:

 * https://github.com/mmun/ember-pathguard

 This RFC would make Torii's intent possible without reverting to patching the RouterDSL proto as is happening here:

 * https://github.com/Vestorly/torii/blob/master/addon/router-dsl-ext.js

### Prior art in other Javascript frameworks

In angular-land, the router config allows providing a `data` object: `The data property ... is a place to store arbitrary data associated with this specific route. The data property is accessible within each activated route. Use it to store items such as page titles, breadcrumb text, and other read-only, static data.`

The router in vue.js supports a similar object, named `props`.  [docs](https://router.vuejs.org/en/essentials/passing-props.html).

In react-router, since Components are routed directly, it would be trivial to extend the component inline. (The RFC author is not an expert in this area. Please comment if you are.)
