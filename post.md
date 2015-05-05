## A brief history

Back in the summer of 2014 Ember has had a great routing and testing story and with Ember CLI, it was on course to have probably the best tool for building front-end tools. There were, however, a few areas where it still lacked a standard way of doing things and animation was one of them.

That all changed when [Edward Faulkner](https://twitter.com/eaf4) started developing liquid-fire which has now become the de facto standard for adding animations to your Ember application.

It has come a long way since then. It's first 1.0 beta version should be out on the 12th of June, the big day for Ember when Ember 2.0 and Ember Data 1.0 should also land.

## Setup

As liquid-fire is an Ember CLI addon, using it in your Ember CLI app is very straightforward:

```
npm install --save-dev liquid-fire
```

Once that is done, we are ready to start adding animations, but let's first see the (simple, yet powerful) architecture of liquid-fire.

## The building blocks of an animation

"Transition", meaning changing from one state to another, is the key concept in liquid-fire. That change can be between two routes, two boolean values or two collections of objects. Transitions can also be animated and in this blog post they will be so I will use the terms "transitions" and "animations" interchangably.

There are three components of an animation in liquid-fire, the *transition map*, the *transition functions* and the *template helpers*.

### Transition map

The transition map is similar to the routing map. Just as the routing map defines the routes of the Ember application, the transition map defines the transitions. The map lives under `app/transitions.js` and has the following syntax:

```javascript
export default function(){
  this.transition(
    this.fromRoute('band.songs'),
    this.toRoute('band.details'),
    this.use('toRight'),
    this.reverse('toLeft')
  );
  (...)
}
```

Each call to `this.transition` defines a rule. Its arguments define constraints (*when* that transition should be triggered) and which animation(s) should be used if all constraints are matched.

In the above example `this.fromRoute` and `this.toRoute` constrain the transition to be activated if the route changes from `band.songs` to `band.details` and specify the `toRight` animation to be used. The last argument, `this.reverse` is a nifty shorthand that says that if the transition happens in the other direction, from `band.details` to `band.songs`, the `toLeft` animation should be employed.

That brings us to our second component, the animation functions.

### Animation functions

The animation functions implement the actual animation from the old state to the new state. Each animation function resides in its own module and exports a single function.

A non-working example would look like this:

```javascript,linenums=true
import { animate } from "liquid-fire";

export default function doubleHelix(opts) {
  var self = this;
  return animate(this.oldElement, params, opts).then(function() {
    return animate(self.newElement, ...);
  });
}
```

`animate` is the main primitive liquid-fire provides to define custom animations. It returns a promise which gives a simple API to order animations (above, animating `newElement` will only begin after oldElement has fully finished animating).

If the above content was stored in `app/transitions/double-helix.js`, it would give us an animation name of `double-helix` that we could use as the argument to `this.use` (and `this.reverse`) in the transition map.

The good news is that a handful of animations come included in the library so we don't need to write any code for implementing animations while we familiarize ourselves with liquid-fire.

### Template helpers

The third and final piece of the puzzle is template helpers. Ember uses Handlebars as its templating language and adds helpers to it to enable certain feature. One example is `link-to` for moving between routes, something that the pure Handlebars language does not support.

However, these Ember helpers don't know anything about animations so "transition-aware" counterparts need to be defined. These have the `liquid-` prefix followed by the name of the Ember helper so we have `liquid-if`, `liquid-with` and `liquid-outlet`, and so on.

Since these helpers observe values passed to them, they can pass the observed change to the transition map. The transition map can then decide whether there is a matching rule and then invoke the corresponding animation.

This architecure is splendidly declarative, decoupled and thus flexible. In the rest of the post, we'll first look at a few basic examples to warm up and then a couple of advanced examples from [Edward's EmberConf talk]((http://www.youtube.com/watch?v=p8aF-7-_cE8) to cap it all off.

## Basic animations

### Animating routing

Let's return to our first example, the one that defined a rule to animate between two routes:

```javascript
// app/transitions.js
export default function(){
  this.transition(
    this.fromRoute('band.songs'),
    this.toRoute('band.details'),
    this.use('toRight'),
    this.reverse('toLeft')
  );
}
```

We saw there are three ingredients for an animation to happen. The first one is defining a rule, and we can check that one out. The second one is implementing an animation function but since `toRight` is built-in, we can scratch that one off the list, too. The third one is enabling the transition in the view layer, or, in other words, making the change transition aware.

Let's assume that the `band` template has the following content:

```handlebars
<!-- app/templates/band.hbs -->
<ul class="nav nav-tabs">
  <li>{{link-to 'Details' 'band.details' model}}</li>
  <li>{{link-to 'Songs'   'band.songs'   model}}</li>
</ul>
<div class="band-info">
  {{outlet}}
</div>
```

The outlet is where sub-routes render their content, so going from `band.songs` to `band.details` triggers a change in that outlet. All we have to do is to use the liquid-fire helper instead of the Ember one:

```handlebars
<!-- app/templates/band.hbs -->

<ul class="nav nav-tabs">
  <li>{{link-to 'Details' 'band.details' model}}</li>
  <li>{{link-to 'Songs'   'band.songs'   model}}</li>
</ul>
<div class="band-info">
  {{liquid-outlet}}
</div>
```

<iframe width="560" height="315" src="https://www.youtube.com/embed/pv9zM1dM9FQ" frameborder="0" allowfullscreen></iframe>

### Animating a value change

We'll now add an animation for when the user starts to edit the description of a band. Let's now start from the other direction and enable the animation in the template first.

The `band.details` route is where the band's description can be edited and the corresponding template looks like the following:

```handlebars
<!-- app/templates/band/details.hbs -->
<div class="panel panel-default band-panel">
  <div class="panel-body">
    (...)
    {{#if isEditing}}
      <div class="form-group">
        {{textarea class="form-control" value=model.description}}
      </div>
    {{else}}
      <p>{{model.description}}</p>
    {{/if}}
  </div>
</div>
```

In edit-mode (when `isEditing` is true), the template renders a textarea with the current description while in display-mode, the description is shown in a paragraph. We want to animate the transition of going from display-mode to edit-mode and thus we turn the `if` into a `liquid-if`:

```handlebars
<!-- app/templates/band/details.hbs -->
<div class="panel panel-default band-panel">
  <div class="panel-body">
    (...)
    {{#liquid-if isEditing class="band-description"}}
      <div class="form-group">
        {{textarea class="form-control" value=model.description}}
      </div>
    {{else}}
      <p>{{model.description}}</p>
    {{/liquid-if}}
  </div>
</div>
```

We also added a `band-description` class that we'll use as a constraint in the transition rule. Time to open the transition map and add a rule that matches the desired transition.

```javascript
// app/transitions.js

export default function(){
  this.transition(
    this.hasClass('band-description'),
    this.toValue(true),
    this.use('fade', { duration: 500 })
  );
);
```

`hasClass` is a contstraint that will filter the matching elements to those that have the passed CSS class (that's why we added the `band-description` class in the previous step). `toValue` can be used in conjuction with helpers that observe a value. Here, the value passed to `liquid-if` is `isEditing` which is a boolean. We want to animate the transition when the user switches to edit-mode, but not when they flip back to display-mode, so we pass `true` as the argument for `toValue`. Finally, we define that the built-in `fade` animation should be used and it should take 500 milliseconds.

Liquid-fire uses [Velocity.js](http://julian.com/research/velocity/) as its animation library and its `animate` function passes parameters through to Velocity.js, like the `{ duration: 500 }` option above.

<iframe width="560" height="315" src="https://www.youtube.com/embed/sjzrsnZxEO4" frameborder="0" allowfullscreen></iframe>

## Going further

This has been really basic stuff so far, so let's up our game and see some flashy animations that will leave you in awe (at least it did me).

I will use examples from [Edward's Physical Design talk](http://www.youtube.com/watch?v=p8aF-7-_cE8) that he [presented at EmberConf]. The repository has been made public and so you can see all of it [here]((https://github.com/ef4/physical-design-demo).

The demo uses [the speakers page of the conference](http://emberconf.com/speakers.html) but made into an Ember application by scraping the data and building an Ember app on top.

![EmberConf Speakers](http://f.cl.ly/items/080n3f2O472O3q0X3N30/emberconf-speakers.png)

### Explode and fly

Both of the animations we'll see is made possible by a built-in transition function, `explode`. It is more of a meta-transition because it allows defining several animations for different elements on the screen which opens up a lot more possibilities.

The first animation rearranges the speaker mugshots when the Shuffle button is clicked. The template that renders the speakers has the following content:

```handlebars
<!-- app/templates/emberconf/speakers.hbs -->
<button class="shuffle" {{action "shuffle"}}>Shuffle</button>

<div class="speaker-icons">
  {{#liquid-with model as |currentModel|}}
    {{#each currentModel as |speaker|}}
      {{#link-to "emberconf.speaker" speaker}}
        <img class="speaker-image-small" alt="{{speaker.name}}"
             src="{{speaker.imageURL}}" data-speaker-id={{speaker.id}} >
      {{/link-to}}
    {{/each}}
  {{/liquid-with}}
</div>
```

Using `liquid-with` with an object argument makes animation possible when the object changes. Wrapping a `liquid-with` around an `each` enables the animation when the *whole collection* observed by the `each` helper changes. It is basically a less powerful version of a future `liquid-each` because it only triggers an animation if the whole collection changes, but not if individual items are added or removed. When [`liquid-each` arrives](https://github.com/ef4/liquid-fire/issues/65) this can even be made more powerful and simpler.

In this case, however, the whole model (i.e the entire list of speakers) is swapped out by the `shuffle` action so we only need to see the other part, the transition rule to understand how the shuffling animation is implemented:

```javascript
<!-- app/transitions.js -->
export default function() {
 this.transition(
    this.childOf('.speaker-icons'),
    this.use('explode', {
      matchBy: 'data-speaker-id',
      use: ['flyTo', { duration, easing: [250, 15] } ]
    })
 );
}
```

The rule will match all children of the `.speaker-icons` container (see the template above) and thus all speaker photos. The `matchBy` given as an option to the `explode` animation will connect each element with a `data-speaker-id` attribute (which is, all speaker `img` tags) from its old (before the shuffle) to its new (after the shuffle) position. The net effect is that you see all speaker photos fly to their new position concurrently:

<iframe width="640" height="315" src="https://www.youtube.com/embed/LC2Ki3st3vE" frameborder="0" allowfullscreen></iframe>

Neat, isn't it?

### Different animations for different elements

We'll finish by analyzing the most complex animation so far wherein different elements animate differently inside the same transition.

If you click on any of the speaker icons, you see that the clicked speaker moves to its new position while all the other ones slide out to the left. The URL also changes since we have transitioned from the speakers page to the detailed page of a speaker:

<iframe width="640" height="315" src="https://www.youtube.com/embed/D_QbBQIW2OQ" frameborder="0" allowfullscreen></iframe>

The template for the detailed speaker page looks something like this (the original code contains components but I unwrapped them for more clarity):

```handlebars
<!-- app/templates/emberconf/speaker.hbs -->

{{link-to "Speakers" "emberconf.speakers" tagName="button" class="speakers-btn"}}

<div class="speakers-list-inner">
  <div class="speakers-image" id="{{speaker.id}}">
    <img class="speaker-image-large" alt="{{speaker.name}}"
         src="{{speaker.imageURL}}" data-speaker-id={{speaker.id}} >
    (...)
  </div>
</div>
```

The important thing to note is that both the speaker list and this page have an `img` tag for the speaker and both have a `data-speaker-id` attribute with the same value for the same speaker. That will make it possible to establish a path along which the animation can be carried out.

Now, let's turn our attention the transition rule for this animation:

```javascript
<!-- app/transitions.js -->
export default function() {

  var duration = 500;

  this.transition(
    this.fromRoute('emberconf.speakers'),
    this.toRoute('emberconf.speaker'),
    this.use('explode', {
      matchBy: 'data-speaker-id',
      use: ['flyTo', { duration } ]
    }, {
      use: ['toLeft', { duration } ]
    }),
    this.reverse('explode', {
      matchBy: 'data-speaker-id',
      use: ['flyTo', { duration } ]
    }, {
      use: ['toRight', { duration } ]
    })
  );
}
```

We are already familiar with the `fromRoute` and `toRoute` constraints, so let's focus on the `explode` transition. This time, `this.use` has two option arguments. The first one match up all the elements that have the same `data-speaker-id` on the old and the new page. Since the new page only has one such element, that one image (the clicked speaker's photo) will fly to its position on the detailed page. The second one matches everything else (all the other speaker photos) and shifts them out to the left.

Finally, a reverse transition is defined. The one matching speaker photo flies back to the list while the other ones slide in from the left (heading to the right).

### Go forth and animate

That concludes our little "up & running" post for animations in Ember.js. Hopefully you now have a solid base for adding animations to your Ember apps.

I recommend checking out [the official documentation](http://ef4.github.io/liquid-fire/), which is, true to the project's awesomeness, an interactive Ember application using liquid-fire.

Before you leave, though, I have a question for you: before liquid-fire, have you ever seen the browser's Back button animate?
