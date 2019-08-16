---
published: true
title: "From Promises to Observables"
cover_image: ""
description: "How I switched my site from a Promises to a Reactive implementation with Observables"
tags: rxjs, javascript, typescript, webdev, angular
series:
canonical_url:
---

Recently I attended the Angular Denver Conference in Denver, Colorado. It was an awesome experience and one of the biggest takeaways I brought home was the power of RxJS.

While I was at the conference I attended an RxJS workshop led by [Jan-Niklas Wortmann](https://twitter.com/niklas_wortmann) and [Michael Hladky](https://twitter.com/Michael_Hladky).  I have previously used RxJS in some of my Angular projects, but wanted to learn more of the fundamentals and really understand how the technology works.  During the workshop I learned how to think in terms of streams, and how the basic fundamentals of RxJS work.  I also learned about the concepts behind Higher Order Observables, and how you can use them to increase performance in your applications.

I was so impressed with RxJS that I went home and actually used Observables to improve performance for one of the pages on my site [andrewevans.dev](https://www.andrewevans.dev).

In this post I'm going to cover how I was able to use RxJS to increase performance in my site.  Specifically I'm going to show how I was able to use RxJS to manage multiple HTTP calls at once, and how this significantly improved my user experience.

I'm also going to go over some basics, but I highly recommend the official RxJS documentation at [rxjs.dev](https://rxjs.dev).  

I've created a small Angular application that showcases what I did.  You can view it on [Stackblitz](https://stackblitz.com/edit/learning-rxjs-with-angular) or at my [GitHub repo (https://github.com/andrewevans0102/learning-rxjs-with-angular).

This post also assumes you have a working knowledge of Angular. The example that I’m going to be showcasing is a traditional promise based approach compared with a reactive approach using RxJS.

# IMPERATIVE VS DECLARATIVE

Two big words you often see with RxJS are __imperative__ and __declarative__.

__Imperative__ refers to code that you have to manually write yourself. This is code that you specifically write to act in a specific way. For synchronous tasks this is perfect, but for handling application events this can be cumbersome.

__Declarative__ refers to letting RxJS do the work for you. Basically, by leveraging the library you define event stream flow.  Instead of having to specifically build code around handling different events, RxJS enables you to use __observables__ and __operators__ to do the work for you.

This all will be easier to understand as we go through the next sections. I'm just introducing these topics first.

# THE BASICS

At its core, RxJS is a library that utilizes streams for handling asynchronous activities. RxJS is a safe way to handle events in your code through the predefined behaviors and contracts that come with the __observables__.

> I know I’m just hitting the high points here, but for a more in depth explanation I highly recommend the documentation at [rxjs.dev](https://rxjs.dev).

RxJS has [observables](https://rxjs.dev/guide/observable), [operators](https://rxjs.dev/guide/operators), and [schedulers](https://rxjs.dev/guide/scheduler). RxJS also makes use of [subjects](https://rxjs.dev/guide/subject) for multicasting events in your applications.

Most people will first encounter RxJS through observables. An observable will typically look something like this:

```ts
import { Observable } from 'rxjs';

const observable = new Observable(function subscribe(subscriber) {
  try {
    subscriber.next(1);
    subscriber.complete();
  } catch (err) {
    subscriber.error(err);
  }
});
```

If you notice there are the following calls:

- __next__
- __complete__ 
- __error__

These are based around the observable model or contract.  __Next__ is what handles emitting events in the stream.  __Complete__ frees up the observables resources and essentially ends the stream.  __Error__ will return an error to anything that has __subscribed__.

What is a subscription?  __Subscriptions__ in RxJS are what starts a stream's execution. Whatever is defined in the __next__ value will emitted as soon as a subscription is started. When a call is made to __complete__, the resources are freed and this observable is essentially finished.

You can also end a stream with __unsubscribe__ or __complete__.  If you use __unsubscribe__, you end a stream manually meaning that the resources are freed and there will be no more events.  If you use __complete__ then it marks the stream as finished.  To clarify, when thinking of __unsubscribe__ and __complete__ just remember:
- __unsubscribe__ means "the stream is not interested in new values"
- __complete__ means "the stream is finished"

When you see __operators__, they are static functions that provide all of these same services we see in __observables__ out of the box. Operators can be intimidating because there is a large number. However, most of them are wrapped around core behaviors.  I highly recommend the workshop I mentioned earlier with [Jan-Niklas Wortmann](https://twitter.com/niklas_wortmann) and [Michael Hladky](https://twitter.com/Michael_Hladky) for a more in depth explanation using what they call the "algebraic approach" to operators.

# MY EXAMPLE
In my example I’m going to use both observables and operators.

The challenge that I wanted to resolve was that the blog page on my site [andrewevans.dev](https://www.andrewevans.dev) required retrieving several RSS feeds.  I originally had coded it to take in all of the HTTP calls to the RSS feeds with the `promise.all()` approach. This basically tried to run all of them as promises in parallel, and when the requests completed then I could return all of the data.  The code in my API endpoint looked like the following:

```js
      const output = [];
      // feed addresses to use in call to rss parser
      let feedInput = [
          {
            sourceURL: "https://medium.com/feed/@Andrew_Evans"
          },
          {
            sourceURL: "https://rhythmandbinary.com/feed"
          },
          {
            sourceURL: 'https://dev.to/feed/andrewevans0102'
          }
      ];
      const promises = [];
      feedInput.forEach((feed) => {
        // add all rss-parser calls as promises
        promises.push(parser.parseURL(feed.sourceURL)
          .then((response) => {
            response.items.forEach((item) => {
              let snippet = '';
              if(item.link.includes('dev.to')) {
                  snippet = striptags(item['content']);
              } else {
                  snippet = striptags(item['content:encoded']);
              }
    
              if(snippet !== undefined) {
                  if(snippet.length > 200) {
                      snippet = snippet.substring(0, 200);
                  }
              }

              const outputItem = {
                sourceURL: feed.sourceURL,
                creator: item.creator,
                title: item.title,
                link: item.link,
                pubDate: item.pubDate,
                contentSnippet: snippet,
                categories: item.categories
              };
              output.push(outputItem);
            })
          })
          .catch((error) => console.log(error))
        );
      });

      try {
        await Promise.all(promises);
      } catch (error) {
        res.status(500).send(error);
      }
```

If you notice the code here basically wraps a set of URLs that are called by the `callRSS` function.  That function just uses the [rss-parser](https://www.npmjs.com/package/rss-parser) to call the RSS feeds and parse the values.  This code looks like the following:

The resulting experience from this code was that (in the Angular client) I had to provide a traditional spinner on the page to show until all of the promises completed.  This actually ended up being several seconds and was not optimal.

In the example, if you go over to the `functions` folder in the `index.js` file you can see the endpoint's code that wraps the promises in the `traditional` endpoint.

In the same example project, if you go over to the `traditional` Angular Component you'll see the client code calling the endpoint with the following:

```ts
  async load() {
    // show spinner while fetching posts
    this.showSpinner = true;

    // retrieve the posts by calling the endpoint that uses promise.all for fetching
    // all of the rss feeds and waiting (synchronously) for them to complete
    this.posts = [];
    const response = await axios.get(environment.traditionalEndpoint);
    response.data.forEach((entry) => {
      const inputDate = new Date(entry.pubDate);
      entry.pubDate = inputDate.toLocaleDateString('en-us') + ' at ' + inputDate.toLocaleTimeString('en-us');
      
      this.posts.push({
        ...entry, 
        sortDate: inputDate.getTime()
      });
    });
    this.posts = response.data;

    // retrieve the manual entries
    const manualEntries: any
      = await axios.get(environment.manualEntries);
    manualEntries.data.forEach((entry: any) => {
      const inputDate = new Date(entry.pubDate);
      entry.pubDate = inputDate.toLocaleDateString('en-us') + ' at ' + inputDate.toLocaleTimeString('en-us');
      if (entry.contentSnippet.length > 200) {
        entry.contentSnippet = entry.contentSnippet.substring(0, 200);
      }

      this.posts.push({
        ...entry, 
        sortDate: inputDate.getTime()
      });
    });

    // sort by date here
    this.posts.sort((a: any, b: any) => {
      return b.sortDate - a.sortDate;
    });

    // stop showing spinner when fetch completes
    this.showSpinner = false;
  }
```

Here I've created a `load` method that uses [axios](https://www.npmjs.com/package/axios) to make a GET call to retrieve the data.  I also call an endpoint for an additional set of manual entries.  When both complete I stop showing the spinner.

# Enter RxJS

So after my experiences from Angular Denver, I started thinking of ways that RxJS could improve this whole setup.  I really didn't like the spinner and several second wait time, so thought this was a great opportunity to improve the site.

I decided that it would help if I could break down the HTTP calls to be handled individually as streams.  Then the user would see results immediately on their page, and it would update as the calls completed.  Since its only a matter of seconds, this didn't make the page jump around too much and made the site feel very responsive.

> I should note that typically with frontend engineering you don't want a page to jump around.  In this situation it was only a matter of seconds and the way that I built the flow made it hardly noticeable.  It was more important for me to show the output immediately.  In a larger application this approach may need to be reworked for a better experience.

I refactored the HTTP calls to be done in one endpoint.  If you look in the example project in the `functions/index.js` file you will see this as the following:

```js
app.get('/reactive/:source', (req, res) => {
  (async () => {
    try{
      const output = [];
      let sourceURL = '';
      if(req.params.source === 'medium') {
        sourceURL = 'https://medium.com/feed/@Andrew_Evans';
      } else if (req.params.source === 'wordpress'){
        sourceURL = 'https://rhythmandbinary.com/feed';
      } else if (req.params.source === 'devto'){
        sourceURL = 'https://dev.to/feed/andrewevans0102';
      }

      const feedOutput = await parser.parseURL(sourceURL)
      feedOutput.items.forEach(item => {
        // when categories is undefined that is normally a comment or other type of post
        if(item.categories !== undefined || item.link.includes('dev.to')) {
          let snippet = '';
          if(item.link.includes('dev.to')) {
              snippet = striptags(item['content']);
          } else {
              snippet = striptags(item['content:encoded']);
          }

          if(snippet !== undefined) {
              if(snippet.length > 200) {
                  snippet = snippet.substring(0, 200);
              }
          }

          const outputItem = {
              sourceURL: sourceURL,
              creator: item.creator,
              title: item.title,
              link: item.link,
              pubDate: item.pubDate,
              contentSnippet: snippet,
              categories: item.categories
          };
          output.push(outputItem);
        }
      });

      res.status(200).send(output);
    } catch(error) {
      res.status(500).send(error);
    }
  })();
});
```

The code here is pretty straightforward, based on the "source" parameter it makes a call to to the matching RSS feed.  The results are gathered from the HTTP call and returned in the output value.

Now for the RxJS implementation, I wrapped each of the HTTP calls to this endpoint in a separate observable.  This enabled each call to stream in at the same time, and as soon as results were found they were stored and shown on the page.

```ts
  load() {
    const medium = 
      this.http.get(environment.reactiveEndpoint + '/medium')
      .pipe(
        catchError(err => {
          throw 'error in source observable. Message: ' + err.message;
        })
      );

    const wordpress = 
      this.http.get(environment.reactiveEndpoint + '/wordpress')
      .pipe(
        catchError(err => {
          throw 'error in source observable. Message: ' + err.message;
        })
      );

    const devto = 
      this.http.get(environment.reactiveEndpoint + '/devto')
      .pipe(
        catchError(err => {
          throw 'error in source observable. Message: ' + err.message;
        })
      );

    const manualEntries = 
      this.http.get(environment.manualEntries)
      .pipe(
        catchError(err => {
          throw 'error in source observable. Message: ' + err.message;
        })
      );

    this.posts$ =
      merge(medium, wordpress, devto, manualEntries)
        .pipe(
          scan((output: Post[], response: []) => {
            response.forEach((post: Post) => {
              const inputDate = new Date(post.pubDate);
              post.pubDate = inputDate.toLocaleDateString('en-us') + ' at ' + inputDate.toLocaleTimeString('en-us');
              post.sortDate = inputDate.getTime();

              if (post.sourceURL === 'https://blog.angularindepth.com/feed') {
                post.sourceURL = 'Angular-In-Depth';
              } else if (post.sourceURL === 'https://itnext.io/feed') {
                post.sourceURL = 'ITNext';
              } else if (post.sourceURL === 'https://medium.com/feed/@Andrew_Evans') {
                post.sourceURL = 'Medium';
              } else if (post.sourceURL === 'https://rhythmandbinary.com/feed') {
                post.sourceURL = 'Rhythm and Binary';
              } else if (post.sourceURL === 'https://dev.to/feed/andrewevans0102') {
                post.sourceURL = 'DEV.TO';
              }
              output.push(post);
            })

            output.sort((a: any, b: any) => {
              return b.sortDate - a.sortDate;
            });

            return output;
        }, []),
        catchError(err => {
          throw 'error in source observable. Message: ' + err.message;
        }),
        takeUntil(this.unsubscribe)
      );
  }
```

Here I am taking advantage of Angular's [HttpClient](https://angular.io/guide/http) which wraps HTTP calls in an observable. 

I then use the [merge](https://rxjs.dev/api/index/function/merge) operator to subscribe to all of the HttpClient calls and combine them into one output.

The [scan](https://rxjs.dev/api/operators/scan) operator then takes the merged observables and appends the response to one common output.

I include the [catchError](https://rxjs.dev/api/operators/catchError) operator to handle any errors in the stream, if one of the calls fails etc.

I also use __pipe__ to take the output of one observable and passes it into another.  This is fairly intuitive and a common pattern with RxJS.

The last operator that was passed into the __pipe__ also references a [takeUntil](https://rxjs.dev/api/operators/takeUntil) operator. This is a very powerful RxJS operator that will unsubscribe an observable based on an event you pass in.  Here I've created a [subject](https://rxjs.dev/guide/subject) which handles unsubscribing this main observable when the code finishes running.  This is a fairly common pattern when handling observables.  RxJS __subjects__ also can be used for multicasting and doing observable like actions.  I'm just using it here because it provides the behavior I wanted and makes a simple `unsubscribe` call clear out the resources.  If I didn't do this it could cause `memory leaks` and potentially freeze up my browser session.  You can see this behavior in the `reactive` component's `clear` method:

```ts
  clear() {
    this.unsubscribe.next();
    this.unsubscribe.complete();
    this.posts$ = null;
  }
```

Also note that I make the observable `null`.  This is not necessary, but for the basic example application I wanted to visually show the data disappearing when `clear` was called.

> As a disclaimer here, there are also 100+ other ways to do this same type of event handling.  Additional implementations include using __from__ to cast promises, __fromFetch__ using the `fetch` API, or even an ajax implementation with `rxjsw`.  I chose the implementation here because it was easy to follow and I was happy with the output.  RxJS is very flexible and gives you the power to control the streams with (almost) any behavior.

You can see this code in the `reactive` Angular Component in my project.

The `load` method does the subscription and starts the stream.

The `clear` method stops the stream and clears the array that is shown on screen.

# Marble Diagrams

The code that I've written here resulted in a streamed approach to the RSS calls that I made.  It made my application more responsive and made it so I didn't need a spinner and wait experience as I had originally.

To understand this behavior, it might be helpful also to have a basic marble diagram.  Marble diagram's are great ways to graphically represent RxJS behavior.  

Here's a Marble Diagram explained:

![marble diagram](./assets/marble-diagram-anatomy.svg)

The following is a copy of the `merge` marble diagram from the RxJS documentation:

![merge operator](./assets/merge.png)

The following is a copy of the `scan` marble diagram from the RxJS documentation:

![scan operator](./assets/scan.png)

To see all this in action, look at my application in stackblitz.  The application flow is very intuitive.  The `traditional` tab makes the HTTP calls in the traditional (imperative) approach, and the `reactive` tab makes the HTTP calls using the RxJS observables and operators I've been discussing.

{% stackblitz learning-rxjs-with-angular %}

# Closing Thoughts

So here I've introduced some RxJS concepts and shown a working example.  

I've shown how you can change your project from using Promises to Observables with RxJS.

Reactive extensions are a big shift in traditional software development.  Streams make our applications more responsive and are actually easier to build.  

I recommend checking out the [RxJS documentation](https://rxjs.dev) and my example project for more.

Hope you enjoyed my post!  Feel fee to leave comments and connect with me on Twitter at [@AndrewEvans0102](https://twitter.com/AndrewEvans0102) and at [andrewevans.dev](https://www.andrewevans.dev).
