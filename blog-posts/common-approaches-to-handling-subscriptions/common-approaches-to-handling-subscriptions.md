---
published: true
title: "Common Approaches to Handling Subscriptions"
cover_image: ""
description: "This blog article covers some common approaches to handling Observable subscriptions with RxJS."
tags: rxjs, javascript, typescript, angular
series:
canonical_url:
---

When developers first start using RxJS one of the biggest challenges is handling subscriptions.

**Subscriptions** in RxJS handle the execution flow of a stream. When we create Observables, we "subscribe" to them to start using them. Conversely, when we "unsubscribe" we do the reverse to stop the same execution flow.

Handling this can be a bit tricky. This post is going to cover some common patterns for handling subscriptions in your code.

This post is also going to be framework agnostic in an effort to make these patterns accessible by all.

The examples that are used in this post can be reached in my [Stackblitz Project](https://stackblitz.com/edit/rxjs-example-handle-susbcriptions).

I'm going to show the code here, and have an embedded link to my Stackblitz project at the end. I encourage you to run the code examples that I walkthrough to get a better understanding.

## Memory Leaks and Your First Unsubscribe

When we do not successfully unsubscribe from an Observable, we create a situation called a "memory leak." This is any time a stream is started (using system resources) and not stopped. If you have enough streams started without an "unsubscribe," you can use up a lot of your system resources and significantly slow down your application..._this is not a good thing_.

A good example of this would be a simple Observable from the creation operator `interval`. Consider the following code:

```javascript
import { interval } from 'rxjs';

const observable = interval(1000);
const subscription = observable.subscribe(() => console.log('Hello!'));
```

So in this example we are just using the [interval](https://rxjs.dev/api/index/function/interval) operator to create a stream that writes "Hello!" to the console every 1 second. When we call `subscribe` we are saying that whenever the stream emits a response (in this case every 1 second), we print "Hello!".

This is very simplistic, but the challenge here is that if we do not call `unsubscribe`, this stream will continue running until you end your session or destroy the associated component etc. This is really easy to miss and important for performance.

To fix this situation, a simple "unsubscribe" is needed. So consider the same code but with the addition of an "unsubscribe" call like so:

```javascript
import { interval } from 'rxjs';

const observable = interval(1000);
const subscription = observable.subscribe(() => console.log('Hello!'));
setTimeout(() => {
  subscription.unsubscribe();
  console.log('unsubscribed');
}, 1000);
```

> NOTE: We are using a `setTimeout` here just to allow for a second to pass and the "hello" to be shown in the console.

Now with the "unsubscribe" called, the execution ends correctly and you're successfully managing the stream.

## Using the take Operator

So in the previous example the subscription was managed manually with direct calls to `subscribe` and `unsubscribe`. This pattern is fine but is also easy to forget to do.

A less error prone approach would be to make use of the `take` operator. When passed into an Observable, the `take` operator enables you to end the execution after a set number of emissions from the stream.

Consider the following code:

```javascript
import { interval } from 'rxjs';
import { take } from 'rxjs/operators';

const intervalCount = interval(1000);
const takeObservable = intervalCount.pipe(take(2));
takeObservable.subscribe(x => console.log(x));
```

When you run this, you should see the following:

```bash
0
1
```

Now what if you changed that same code to the following:

```javascript
import { interval } from 'rxjs';
import { take } from 'rxjs/operators';

const intervalCount = interval(1000);
const takeObservable = intervalCount.pipe(take(10));
takeObservable.subscribe(x => console.log(x));
```

When you run this, you should see the same as before but the count goes from 0 to 9.

So what's happening? The take operator just controls the execution flow so that the number you pass in determines how many times it emits a value before completing. You don't have to worry about a memory leak here because the completion formally stops the execution flow here.

In addition to the `take` operator there are multiple other examples of ways to do this behavior.

Some include the following:

- [takeWhile](https://rxjs.dev/api/operators/takeWhile)
- [takeUntil](https://rxjs.dev/api/operators/takeUntil)
- [takeLast](https://rxjs.dev/api/operators/takeLast)
- [first](https://rxjs.dev/api/operators/first)

The important thing about this behavior is just that you're letting RxJS handle the stream for you. This lets you write clean code that is easily maintainable.

## Combining Subscriptions

Another common pattern that you run across is when you multiple observables, and want to manage their subscriptions together.

Consider the following code:

```javascript
import { Subscription, of } from 'rxjs';

// create a subscription object
const subs = new Subscription();

// create observables
const value$ = of(1, 2, 3, 4);
const anotherValue$ = of(true);

// subscribe to observables and add to subscription
subs.add(value$.subscribe(x => console.log(x)));
subs.add(anotherValue$.subscribe(x => console.log(x)));

// calling subs.unsubscribe() will unsubscribe from all sub
subs.unsubscribe();
```

> special thanks to my friend [Tim Deschryver](https://twitter.com/tim_deschryver) for this specific example

In this example you see that we define an instance of a [Subscription](https://rxjs.dev/api/index/class/Subscription) that we add two observables to. The `Subscription` class enables you to wrap your subscriptions in one resource. When you're ready to dispose of your application, you can just call a singular `unsubscribe` and execution in all of the observables wrapped will be properly stopped.

This pattern is particularly useful when you have multiple observables in a component that you want to manage together. It makes the implementation cleaner and easier to maintain.

## Combining Subscriptions with tap and merge

In addition to the above example, another common pattern is to make use of the [tap](https://rxjs.dev/api/operators/tap) operator and static [merge](https://rxjs.dev/api/index/function/merge) function to combine multiple observables.

Consider the following code:

```javascript
// create observables
const value$ = of(1, 2, 3, 4).pipe(tap(x => console.log(x)));
const anotherValue$ = of(true).pipe(tap(x => console.log(x)));

const subs = merge(value$, anotherValue$).subscribe();

subs.unsubscribe();
```

The static [merge](https://rxjs.dev/api/index/function/merge) function enables you to combine many observables into a single value. Then when you are ready to stop execution, a single `unsubscribe` stops execution on the group. This pattern is very clean, and enables RxJS to handle the orchestration of your streams without needing to declare additional operators etc.

## Closing Thoughts

So in this post you saw a few patterns for handling subscriptions with RxJS. The really great part is that RxJS is very flexible and can accommodate (almost) any use-case. I hope the examples here have provided you with some basic patterns for your applications. Feel free to leave comments and follow me on Twitter at [@AndrewEvans0102](https://twitter.com/andrewevans0102)!

Here's a stackblitz for the examples above:
{% stackblitz rxjs-example-handle-susbcriptions %}
