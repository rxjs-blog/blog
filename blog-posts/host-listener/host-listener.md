---
published: true
title: "Ain't nobody needs HostListener"
cover_image: ""
description: "In Angular event handling is often implemented using the hostListener decorator, even though it might not be the best fit for the problem. Have you considered more composable approaches?"
tags: angular, rxjs, javascript, typescript
series:
canonical_url:
---

Angular's `@hostListener` is well known within the community. Rather unknown are the problems this might have on runtime performance and general application architecture. In general, there are three main problems with using the `hostListener` decorator.

1. Missing composability
2. Performance issues
3. Lacks configuration options

Before tackling those two problems more in detail, let's have a look at the example code used to demonstrate the problem.
To do so let's have a look at the following Stackblitz example, particularly the `BoxComponent`: 

{% stackblitz angular-hostlistener-imperative file=src/app/box.component.ts %}

Here we see an implemented drag'n'drop feature, using the `@hostListener` decorator. In total, we registered 3 listeners. - A `mousedown` event, which we are using to set a property signalling that our drag'n'drop is about to start. Afterwards,- A `mousemove` event, which calculates the position of the rectangle according to the mouse position.
- Finally, we are using the `mouseup` event to signal that our drag'n'drop has ended.

Do note that we used `document` as eventTarget. We needed that to handle fast mouse movements which might be out of sync with the position of the rectangle. One will notice that when moving the mouse very fast, that one is out of the rectangle element, which would stop our drag'n'drop.

## Problems

Let's have a more in-depth look at the problems listed above.

### Missing composability

Taking a look into the code, we will notice that we set the property `isClicked` to `true` as soon as the `mousedown` event happens. We use that property to perform an early return inside of the `mousemove` event handler to stop this function from execution. This is the only way we can compose those two events, which is quite expensive because this `mousemove` function is still executed with every mouse movement. In terms of composition, this drag'n'drop feature is fairly straight forward. There are several much more complex event composition scenarios, which become extremely difficult when using the `@hostListener` decorator.  

### Performance issues

This problem is mostly the resolution of the missing composability. The problem here is that we register the 3 event listener, mentioned above, for every component instance, even though it's impossible to drag'n'drop multiple rectangles at the same time. Therefore what we should aim for is, that only the `mousedown` event listener is registered for every component and just when this event happens we register the other events accordingly. Doing all this logic within the event listener function is a lot of work and also decently complex. Additionally, there is currently no way to disable a registers `@hostListener` function. This is also the reason why the code example above constantly listens to mouse move events, even though they are not relevant if there isn't a rectangle selected before. 

### Lacks configuration options

Usually, the [`addEventListener`](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/addEventListener) provides an argument for configuration options (the description below is copied from the MDN web docs):
 - **capture:** A `Boolean` indicating that events of this type will be dispatched to the registered `listener` before being dispatched to any `EventTarget` beneath it in the DOM tree. 
 - **once:** A `Boolean` indicating that the `listener` should be invoked at most once after being added. If `true`, the `listener` would be automatically removed when invoked.
 - **passive:** A `Boolean` which, if `true`, indicates that the function specified by `listener` will never call `preventDefault()`. If a passive listener does call preventDefault(), the user agent will do nothing other than generating a console warning.

 One can clearly see that those configuration options are very powerful. For sure, one probably don't need to use them for every case. But especially for heavily event-oriented features this configuration options are key. If we take a look at the [offical Angular documentation](https://angular.io/api/core/HostListener) we will see, that we are not able to specify these configuration parameters, when using the `hostListener` decorator.  

## Alternative Approaches

We have two different approaches to tackle the problems described above. Depending on your knowledge some of them are more or less complex. Let's have a look!

### Using addEventListener

Theoretically one could register nested event listeners. Therefore we could use the `addEventListener` function to register the event listeners.

{% stackblitz angular-hostlistener-eventlistener file=src/app/box.component.ts %}

Looking at the code example one will notice that this is fairly complex. Especially because we need to take care of registering and unregistering the nested event listeners. Even if all of the problems described above can be solved with this approach, In my personal opinion, I think that this is a very complex and hard to understand solution. 


### Using fromEvent

The second alternative approach would be using the RxJS [`fromEvent`](https://rxjs.dev/api/index/function/fromEvent) operator. RxJS shines when it comes to composition of event-oriented code. 

{% stackblitz angular-hostlistener-reactive file=src/app/box.component.ts %}

Having a look at this code, one will notice that just looking at the lines of code that this is the smallest approach. I have to admit that one needs to be familiar with RxJS to understand and write such code. It's not really intuitive, but therefore RxJS takes care of registering and unregistering the event listener for us. Additionally, we have many more opportunities regarding composability. That's one of the key benefits of using RxJS when dealing with event-oriented code. 

If you want to understand the used operators you can have a look at the following blog posts:
  - [switchMapTo](https://dev.to/rxjs/about-switchmap-and-friends-2jmm)
  - [takeUntil](https://dev.to/rxjs/takewhile-takeuntil-takewhat-5006) 

## Summary

The `@hostListener` decorator is handy if we just want to listen to single events and don't rely on any kind of composition. Everything that involves a certain event composition should be implemented by using one of the other approaches listed above. In general, `@hostListener` lacks features that are necessary when dealing with event composition. It completly misses *cancellation* options and any kind of *composability*. Those features are crucial when building heavily event oriented features.
When you are used to RxJS you should probably use the `fromEvent` operator to perform any kind of complex event handling. If RxJS is not your preferred technology, maybe using plain old `addEventListener` might be a viable option for you.

## Disclaimer

This blog post aims to elaborate on different approaches to deal with event composition. It never intends to blame or hurt someone who was involved in the design or implementation of the `@hostListener` feature. I personally appreciate any work that was put into that. 
