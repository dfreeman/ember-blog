---
title: Ember 1.0 RC8 Released
authors:
  - tom-dale
  - yehuda-katz
  - peter-wagenet
  - alex-matchneer
date: 2013-08-29T00:00:00.000Z
tags:
  - releases
  - version-1-x
  - '2013'
---


<!--alex disable just-->
With Ember 1.0 RC8, we have reached the final RC before 1.0 final, which
we hope to release this weekend if all goes well.

This final release candidate comes with a couple of small breaking
changes that we are making before 1.0 because they should have a small
impact on application code but a large impact on performance.

We normally would not make changes of this nature this close to the 1.0
final release date. However, the performance improvements were so
dramatic we did not want to wait until Ember.js 2.0 to introduce the
change, and we are serious in our commitment to not breaking the API
post-1.0. This was our last chance to get these changes in before
putting the final stamp on the 1.0.

Both of the changes are related to observers. If you find yourself
writing code with lots of observers, you may be writing non-idiomatic
code. In general, you should only need to use observers when you are
bridging Ember code to other libraries that do not support bindings or
computed properties.

For example, if you were writing a component that wrapped a jQuery UI
widget, you might use an observer to watch for changes on the component
and reflect them on to the widget.

In your application code, however, you should be relying almost entirely
on computed properties.

If you are having trouble upgrading your application, please join us in
the IRC channel or post on StackOverflow for help.

#### Declarative Event Listeners

There is now a way to declaratively add event listeners to Ember
classes. This is easier than manually setting up the listeners in
`init`.

Instead of:

```javascript
App.Person = DS.Model.extend({
  init: function() {
    this.on('didLoad', this, function() {
      this.finishedLoading();
    });
  },

  finishedLoading: function() {
    // do stuff
  }
});
```

You can just do this:

```javascript
App.Person = DS.Model.extend({
  finishedLoading: function() {
    // do stuff
  }.on('didLoad')
});
```

#### Array Computed

Thanks to the tremendous work of David Hamilton, there is now a
convenient and robust way to build a computed property based on an array
that will only perform calculations on the updated portion.

For example, say that you have an array of people, and you want to
create a computed property that returns an Array of their ages.

Currently, the easiest way to do that is:

```javascript
App.Person = Ember.Object.extend({
  childAges: function() {
    return this.get('children').mapBy('age');
  }.property('children.@each.age')
});
```

This is pretty terse, but it must recalculate the entire array any time
a single element is added or removed. This works OK for small arrays,
but for larger arrays, or when these kinds of computed properties are
chained or doing expensive work, it can add up.

Enter Array Computed properties:

```javascript
App.Person = Ember.Object.extend({
  childAges: Ember.computed.mapBy('children', 'age')
});
```

You can also chain these Array computed properties together:

```javascript
App.Person = Ember.Object.extend({
  childAges: Ember.computed.mapBy('children', 'age'),
  maxChildAge: Ember.computed.max('childAges')
});
```

When an element is added or removed, the computation is only done once.
In this example, if a child is added, their age is appended to
`childAges`, and if that age is larger than the `maxChildAge`, that
property is updated.

These computed properties are always up to date, efficient, and
completely managed by Ember.

#### Ember Extension

After months of testing, and tons of work by Teddy Zeenny, we're finally
ready to put the Ember Inspector in the Chrome Web Store.

Most recently, Teddy added support for loaded data. It already has
support for Ember Data, and Ember Model is actively working on adding
support.

<img src="/images/blog/rc8-ember-data.png">

Teddy has also significantly improved the object inspector, adding
support for objects to group properties as they wish (e.g. attributes,
has many, belongs to in Ember Data) and editing records in the inspector
itself.

<img src="/images/blog/rc8-editing.png">

You can also see a list of all routes in your app, along with the naming
you should use for associated objects. This should make remembering the
naming conventions a lot easier.

<img src="/images/blog/rc8-routes.png">

And finally, a view tree that shows an overlay in your app with the
associated controller and model for your application's templates.

<img src="/images/blog/rc8-view-tree.png">

It should be in the web store in the next day or so, so keep an eye out.
Follow @emberjs on Twitter to get the latest!

#### Release Cycle

We know that the Ember 1.0 RC cycle was a **bit** long. In truth, we
released RC1 too early. We're sorry if the release names caused
confusion.

Going forward, we plan to seriously tighten up our release process. We
will have a blog post outlining the new process together with the final
Ember 1.0 release.

#### Other Improvements

* Several improvements to `yield` to make sure it always yields back to
  the calling context [@kselden]
* Performance improvement to range updates by not using the W3C range
  API even if it's available [@eviltrout]
* Completion of the 1.0 documentation audit [@trek]
* Better error message if you try to use the same template name multiple
  times by using `<script>` tags [@locks]
* Add `currentRouteName` to `ApplicationController`, which you can use
  in `link-to`, `transitionTo`, etc. [@machty]
* Alias `linkTo` to `link-to` and `bindAttr` to `bind-attr` for
  consistency with HTML naming. Old names remain but are soft-deprecated
  [@ebryn]

#### Changes TL;DR

##### Observers Don't Fire During Construction

Previously, observers would not fire for properties passed into
`create` or specified on the prototype, but they would fire if you `set`
a property in `init`.

Now, observers **never** fire until after `init`.

If you need an observer to fire as part of the initialization process,
you can no longer rely on the side effect of `set`. Instead, specify
that the observer should also run after `init` by using
`.on('init')`.

```javascript
App.Person = Ember.Object.extend({
  init: function() {
    this.set('salutation', "Mr/Ms");
  },

  salutationDidChange: function() {
    // some side effect of salutation changing
  }.observes('salutation').on('init')
});
```

##### Unconsumed Computed Properties Do Not Trigger Observers

If you never `get` a computed property, its observers will not fire even
if its dependent keys change. You can think of the value changing from
one unknown value to another.

This doesn't usually affect application code because computed properties
are almost always observed at the same time as they are fetched. For
example, you get the value of a computed property, put it in DOM (or
draw it with D3), and then observe it so you can update the DOM once the
property changes.

If you need to observe a computed property but aren't currently
retrieving it, just `get` it in your `init` method.

##### New Actions Hash for Routes, Controllers, and Views

Previously, you could define actions for your routes in their `events` hash. In controllers and views however, actions were defined directly as methods on the instance. Not only was this inconsistent, it also lead to awkward naming conflicts. For instance, attempting to name your action `destroy` would conflict with the built-in `destroy` method and very bad things would happen.

To make things consistent and give more flexibility in action naming, we've standardized around using a hash called `actions`. When extending a class with `actions` defined, we'll merge the `actions` defined on the subclass or instance with those on the parent. We also support `_super` so you won't lose any flexibility with this approach.

The old behavior will continue to work for the time being but is now deprecated. In the event that you had a controller that was proxying to a model with an existing `actions` property, we internally rename the property to `_actions` to avoid any conflicts.

##### Enforce Quoted Strings in Handlebars Helpers

In the past, we were loose with our requirements for quoting strings in Handlebars helpers. Unfortunately, this meant we were unable to distinguish between string values and property paths. We are now strictly enforcing quoting if you want the value to be treated as a string. This means that for `link-to` the route names must be quotes. In the reverse direction, if you had custom bound helpers and were passing a property path as a quoted string, this will no longer work. Again, quotes for strings, no quotes for paths.

#### Setting Properties in `init`

Currently, there is an inconsistency between properties set when passing
a hash to `create` and setting those same properties in `init`.

```javascript
App.Person = Ember.Object.extend({
  firstNameDidChange: function() {
    // this observer does not fire
  }.observes('firstName')
});

App.Person.create({ firstName: "Tom", lastName: "Dale" });
```

In this case, because the properties were set by passing a hash to
`create`, the observers are not fired.

But let's look at what happens in RC7 when the same initialization is
done via the `init` method:

```javascript
// WARNING: OLD BEHAVIOR

App.Person = Ember.Object.extend({
  init: function() {
    if (!this.get('firstName')) {
      this.set('firstName', "Tom");
    }
  },
  firstNameDidChange: function() {
    // this observer fires
  }.observes('firstName')
});

App.Person.create({ lastName: "Dale" });
```

In this case, the old behavior would trigger the observers if
`firstName` was not provided.

We intended the design of the object model to trigger observers only
after construction, which is why `create` behaves the way it does.

Also, because the only way to define initial properties that have arrays
or objects as values is in `init`, there is a further inconsistency:

```javascript
// WARNING: OLD BEHAVIOR

App.Person = Ember.Object.extend({
  // initial property value, does not trigger an initialization observer
  salutation: "Mr.",

  init: function() {
    // also initial property value, triggers an observer on
    // initialization
    this.set('children', []);
  }
});
```

In short, properties that get set during initialization, whether they
were already set on the prototype, passed as a hash to `create`, or set
in `init`, do not trigger observers.

If you have some code that you want to run both on initialization and
when a property changes, just mark it as a method that should also run
when initialization is done by using `.on('init')`. This will also be
more resiliant to refactoring, and not rely on a side effect of an
`init`-time `set`.

```javascript
App.Person = Ember.Object.extend({
  firstNameDidChange: function() {
    // some side effect that happens when first name changes
  }.observes('firstName').on('init')
});
```

#### Computed Property Performance Improvements

The latest release of Ember.js contains a change to the way observers and
computed properties interact. This may be a breaking change in apps that
relied on the previous behavior.

To understand the change, let's first look at an example of a computed
property.  Imagine we are trying to model [Schrödinger's famous
cat](http://en.wikipedia.org/wiki/Schr%C3%B6dinger's_cat) using an Ember.js
object.

```javascript
App.Cat = Ember.Object.extend({
  isDead: function() {
    return Math.rand() > 0.5;
  }.property()
});

var cat = App.Cat.create();
```

Given this cat object, is it alive or is it dead? Well, that's determined at random.
Before observing the cat, we might say that it's *both* alive *and* dead, or
perhaps neither.

In reality, whether or not our cat has shuffled off this mortal coil is only
determined the first time we ask for it:

```javascript
cat.get('isDead');
// true
// …or false, half the time
```

After we have asked the cat object for its `isDead` property, we can
categorically say which it is. But before we ask, the value of the
computed property doesn't really _exist_.

Now, let's introduce observers into the mix. If the value of a computed
property doesn't exist yet, should observers fire if one of its
dependent keys changes?

In previous versions of Ember.js, the answer was "yes." For example,
given this class:

```javascript
App.Person = Ember.Object.extend({
  observerCount: 0,

  fullName: function() {
    return this.get('firstName') + ' ' + this.get('lastName');
  }.property('firstName', 'lastName'),

  fullNameDidChange: function() {
    this.incrementProperty('observerCount');
  }.observes('fullName')
});
```

Changing any of the dependent keys would trigger the observer:

```javascript
// WARNING: OLD BEHAVIOR DO NOT RELY ON THIS

var person = App.Person.create({
  firstName: "Yehuda",
  lastName: "Katz"
});

person.get('observerCount'); // => 0

person.set('firstName', "Tomhuda");
person.get('observerCount'); // => 1

person.set('lastName', "Katzdale");
person.get('observerCount'); // => 2
```

However, because the `fullName` property doesn't really "exist" until
it's requested, it is unclear if firing an observer is the correct
behavior.

A related problem affects computed properties if their dependent
keys contain a path. (Remember that dependent keys are just the property
names you pass to `.property()` when defining a CP.)

For example, imagine we are building a model to represent a blog post
that lazily loads the post's comments if they are used (in a template,
for instance).

```javascript
App.BlogPost = Ember.Object.extend({
  comments: function() {
    var comments = [];
    var url = '/post/' + this.get('id') + '/comments.json');

    $.getJSON(url).then(function(data) {
      data.forEach(function(comment) {
        comments.pushObject(comment);
      });
    });

    return comments;
  }.property()
});
```

Awesome! This will work as expected—a post's comments will only be loaded
over the network the first time we do `post.get('comments')` or use it
in a template:

```handlebars
<ul>
{{#each comments}}
  <li>{{title}}</li>
{{/each}}
</ul>
```

However, now we want to add a computed property that selects the first
comment from the array of loaded comments:

```javascript
App.BlogPost = Ember.Object.extend({
  comments: function() {
    var comments = [];
    var url = '/post/' + this.get('id') + '/comments.json';

    $.getJSON(url).then(function(data) {
      data.forEach(function(comment) {
        comments.pushObject(comment);
      });
    });

    return comments;
  }.property(),

  firstComment: function() {
    return this.get('comments.firstObject');
  }.property('comments.firstObject')
});
```

Now we have a problem! Because the `firstComment` computed property has
a dependency on `comments.firstObject`, it will `get()` the `comments`
property in order to set up an observer on `firstObject`.

As you can see, just adding this computed property now means that the
comments are loaded for *all* blog posts in the app—whether their
comments are ever used or not!

To determine what to do, we spent some time looking at real-world
Ember.js apps. What we discovered is that **this behavior carried with
it signficant performance penalties.**

1. Firing observers on unmaterialized computed properties means we have
   to set up listeners on all computed properties ahead of time, instead
   of lazily the first time they are computed.
2. Many computed properties that never get used are nonetheless computed
   because of path dependent keys.

To fix these issues, **RC8 makes the following changes**:

1. Observers that observe a computed property only fire after that
   property has been used at least once.
2. Observing a path (`"foo.bar.baz"`), or using a path as a dependent key,
   will not cause any parts of the path that are uncomputed to become
   computed.

The majority of Ember.js applications should not be affected by this
change, since:

1. Most apps that observe computed properties also `get()` those
   properties at object initialization time, thus triggering the correct
   behavior.
2. In the case of computed property dependent keys, the new behavior is
   what developers were expecting to happen all along.

If your applications are affected by this change, the fix is
straightforward: just `get()` the computed property in your class's
`init` method.

For example, to update the observer example above, we can retain the
pre-RC8 behavior by "precomputing" the `fullName` property:

```javascript
App.Person = Ember.Object.extend({
  init: function() {
    this.get('fullName');
    this._super();
  },

  observerCount: 0,

  fullName: function() {
    return this.get('firstName') + ' ' + this.get('lastName');
  }.property('firstName', 'lastName'),

  fullNameDidChange: function() {
    this.incrementProperty('observerCount');
  }.observes('fullName')
});
```

#### `link-to` Bound Parameters

The `link-to` helper (formerly `linkTo`, see above) now treats
all unquoted parameters (and non-numeric parameters)
as bound property paths, which means that when a property passed to
`link-to` changes, the `href` of the link will change. This includes
the first parameter (the destination route name) and any of the context
parameters that follow.

The following example template will look up the `destinationRoute` on the
current context (usually a controller) and use it to determine the
`href` of the link and the route that will be transitioned to when
the link is clicked.

```handlebars
{{#link-to destinationRoute}}Link Text{{/link-to}}
```

The following example template will always point to the `articles.show`
route (since the route name parameter is in quotes), but when the value
of `article` changes, the link's `href` will update to the URL that
corresponds to the new value of `article`.

```handlebars
{{#link-to 'articles.show' article}}Read More...{{/link-to}}
```

This might cause a few surprises in your app if you haven't been
distinguishing between quoted strings and property paths, so make sure
that any static string `link-to` parameters (such as route names) are
properly quoted in your templates when you upgrade to RC8.

#### Bound Helpers: Quoted Strings, Numbers, and Paths

Invoking a custom bound helper (e.g. one defined via `Ember.Handlebars.helper`)
with quoted strings or raw numbers will pass that raw value directly
into the helper function you've defined, rather than treating everything
like a bound property path that will re-render the helper when changed.

```handlebars
Pass the string 'hello' to myHelper:
{{myHelper 'hello'}}

Pass the property pointed-to by the path 'hello' to myHelper:
{{myHelper hello}}
```

This might cause a few surprises in your app if you've been invoking
bound helpers with quoted strings and expecting them to be treated as
bound property paths, so make sure that the only time you're passing
quoted strings to custom helpers is when you really intend to pass raw
strings (rather than the values of properties) to the helper.
