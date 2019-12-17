---
layout: post
title:  "`this` in Javascript: finally an explanation that makes sense"
categories: web javascript
---

> Is `this` the real life?
>
> Is `this` just fantasy?

\- _Freddie Mercury, while trying to learn Javascript_

# One simple rule, a few exceptions

The one rule to know, is that `this` is _always_ whatever is before the dot (`.`) when you **call** the function.

e.g: `dog.bark()`. Inside the `bark()` function, `this` is going to be the object `dog`, because that's what is before the dot.

**That's it.**

Let's explore <span title="badum-tsss (pun intended)">this</span> in more detail.

# What about the global context?!?!

The global execution context, A.K.A. anywhere outside a function, still abides by this rule. It does so in a roundabout way, but it's obvious once you think about it: `doSomething()` is _exactly the same_ as `window.doSomething()`. So what's before the dot? `window`, of course!

Note: it's `global` instead of `window` in NodeJs.

Now let's explore actual exceptions to the rule.

# Exception the first: `bind`, `apply` and `call`

`bind`'s main purpose is to "set" `this` permanently. So that exception is kind of obvious, and I'll leave it at that. `apply` and `call` both take "thisArg" argument, which will explicitely set `this` for you. Again a pretty obvious exception.

# Exception the second: arrow function

Arrow functions (`() => {}`) are this new concept in ES6. It's been a few years so you should probably know about them by now, but one of their main properties is that they're auto-bound to `this` when they're declared. In other words, it's as if you did:
```javascript
function myFunction() {
  // ...
}
myFunction = myFunction.bind(this);
```

Again, this is one of the main reasons why you'd use arrow functions, so it's not exactly surprising.

# Exception the third: strict mode

Strict mode removes the roundabout equivalence I mentioned earlier with `doSomething()` and `window.doSomething()`. In the first case, `this` will be `undefined` and in the second case it'll still be what's before the dot, so `window`.

# Exception the fourth (and last): constructors

This is the least surprising exceptions to practitioners of a more traditional object-oriented language. `this` inside a constructor (a function invoked with `new`) is the object you're creating.

# Event handlers

Event handlers have a special mention in the [MDN documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this) and in most explanations I've seen online, but they don't really need it because they still follow the same basic rule. When you click on your webpage, this will happen somewhere deep inside your browser:
```javascript
const element = getTheElementYouClickedOn();
element.onclick(); // <-- This is how your function actually gets called
```
So what's the value of `this` inside your event handler? You guessed it, it's the `element`.

# Conclusion

So, by just remembering this simple rule (`this` is _always_ what comes before the dot when you call the function) and 4 exceptions, you'll be able to say "YES!" the next time someone asks:
<iframe width="560" height="315" src="https://www.youtube.com/embed/DJ6CcEOmlYU" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

