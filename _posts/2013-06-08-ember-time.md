---
layout: post
title: 'Ember Time'
---

<iframe
  width="178" height="24" style="border:0px"
  src="http://mixonic.github.io/ember-community-versions/2013/06/08/ember-time.html">
</iframe>

*This article is now seriously out of date. Please refer to [Ember’s official guides](http://guides.emberjs.com/).*

*What follows is a re-publication of a [README][repo].*

The README in question turned out a little better-written and more
instructive than I had expected so I figured it deserved to be published
as a proper little piece.

If you think this is shameless recycling of material, feel free to troll
me on [Twitter][twitter].

---

A friend recently asked about the best approach for implementing a
particular feature in Ember. They wanted to use moment.js to show createdAt
times in ‘time ago’ format -- and wanted them to update each minute.

This is a common feature, but it’s not covered in Ember’s guides and
appears to fall slightly outside of the golden path. So let’s try
implementing it, and hopefully we’ll get a chance to use some of the
lesser-known tools in Ember’s API.

First up, let’s figure out how we want to use this in our templates.
Let’s say we want a `fromNow` helper to match Moment’s API. Let’s add that
to our application template:

{% raw %}
```html
<script type="text/x-handlebars">
  <p>Created {{fromNow valueBinding="createdAt"}}</p>
</script>
```
{% endraw %}

If we reload the page we get an error telling us that `fromNow` is not
defined anywhere. So let’s define it.

This feature involves continued behaviour, so we probably don’t want a
one-shot helper. Instead, let’s have our helper delegate to a taylored
view class.

```javascript
Ember.Handlebars.helper('fromNow', App.FromNowView);
```

Now to define that view class. We probably want it to use the `time`
tag and render the result of running Moment’s `fromNow` method on
whatever its value is.

{% raw %}
```javascript
App.FromNowView = Ember.View.extend({
  tagName: 'time',

  template: Ember.Handlebars.compile('{{view.output}}'),

  output: function() {
    return moment(this.get('value')).fromNow();
  }.property('value')
});
```
{% endraw %}

Note that we use `view.output` rather than simply `output`. This is
because Ember’s views try their best to get out of the way of the
surrounding context. If we want to access a property
of the view in it’s template, we need to be specific.
Also note that the view class will need to be defined above before
the helper, as it references it directly.

If we refresh the page now, we’ll actually see the reasonable output
of ‘Created a few seconds ago’. This is because our view’s value is
not defined and `moment(undefined)` creates a moment object for the
current time.

Let’s define `createdAt` so it’ll become the value of our view.

We’re rendering the view in the application template, which is backed
by the singleton instance of our `ApplicationController`, so let’s
set `createdAt` there:

```javascript
App.ApplicationController = Ember.Controller.extend({
  createdAt: new Date(2011, 3, 30)
});
```

Refreshing the page should show something like ‘Created 2 years ago’.
This is great, but not so good for our demo, so let’s say that
`createdAt` is set to the current time when the app is booted.

```javascript
App.ApplicationController = Ember.Controller.extend({
  createdAt: new Date()
});
```

We’re back to our ‘Created a few seconds ago’ output, but we know
everything’s bound together properly now. It’s time to make this clock
tick.

Our `FromNowView` probably needs some sort of `tick` method to trigger
the re-render. Digging into Ember’s API docs reveals some very helpful
methods in the `Ember.run` namespace. We’ll also need to start this
clock ticking, so let’s use the `didInsertElement` hook.

```javascript
App.FromNowView = Ember.View.extend({
  // ...

  didInsertElement: function() {
    this.tick();
  },

  tick: function() {
    Ember.run.later(this, function() {
      console.log('tick');
      // Re-render the view somehow
      this.tick();
    }, 1000);
  }
});
```

If we open the javascript console now, we should see ‘tick’ written to
the log every second. That’s a start, now we need to figure out how to
re-render the view. Digging again into Ember’s API docs, we find a method
called `notifyPropertyChange` on `Ember.View`. That sounds like it might
work. Let’s give it a go.

```javascript
App.FromNowView = Ember.View.extend({
  // ...

  tick: function() {
    Ember.run.later(this, function() {
      console.log('tick');
      this.notifyPropertyChange('value');
      this.tick();
    }, 1000);
  }
});
```

Leave the page for 60 seconds and we should see ‘Created a few seconds ago’
automatically update to ‘Created a minute ago’ and so on.

This is a good start, but there’s a little problem — there’s nothing
to clean up our `tick` method. If we switch states away from this template
`tick` might get called when the view is no longer on display and we’ll
get errors, not to mention memory leaks.

Digging into Ember’s docs again, we find views have a `willDestroyElement`
hook and Ember provides `Ember.run.cancel` to cancel deferred execution.
So we’ll need to keep a handle on our deferred tick execution and be sure
to cancel it when the view is destroyed.

```javascript
App.FromNowView = Ember.View.extend({
  // ...

  tick: function() {
    var nextTick = Ember.run.later(this, function() {
      console.log('tick');
      this.notifyPropertyChange('value');
      this.tick();
    }, 1000);
    this.set('nextTick', nextTick);
  },

  willDestroyElement: function() {
    var nextTick = this.get('nextTick');
    Ember.run.cancel(nextTick);
  }
});
```

To check this has all worked, let’s rearrange the app a bit. We’ll
create a `clock` route that contains our `fromNow` helper, and jump
back to the index route to check `tick` is not still getting invoked.
We’ll also need to move the value of `createdAt` to a new
`ClockController`.

```javascript
App.Router.map(function() {
  this.route('clock');
});

// ...

App.ClockController = Ember.Controller.extend({
  createdAt: new Date()
});
```

{% raw %}
```html
<script type="text/x-handlebars">
  <h1>Ember Time</h1>

  <nav>
    {{#linkTo index}}Home{{/linkTo}}
    {{#linkTo clock}}Clock{{/linkTo}}
  </nav>

  {{outlet}}
</script>

<script type="text/x-handlebars" data-template-name="clock">
  <p>Created {{fromNow valueBinding="createdAt"}}</p>
</script>
```
{% endraw %}

If everything’s worked, now when we navigate to ‘Clock’ we should see
‘Created a few seconds ago’ and if we leave the app in this state long
enough we’ll see ‘Created a minute ago’. We should also see the console
logging ‘tick’ every second and—all being well—when we navigate back
to ‘Home’ we’ll see the console stops logging ‘tick’.

You can find all of this put together in the [Ember Time Repo][repo] and
[very basic demo][demo].

I hope this little tale of Ember development proves useful to someone
out there. If you’ve read through all this and still have a few minutes
to spare, I recommend [this inspirational video][mamba-time].

[repo]: https://github.com/jgwhite/ember-time
[twitter]: http://twitter.com/jgwhite
[demo]: http://jgwhite.co.uk/ember-time
[mamba-time]: http://youtu.be/5kgUL_FfUZY?t=1h1m13s
