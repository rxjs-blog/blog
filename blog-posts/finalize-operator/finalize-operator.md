# The Finalize Operator

I was building an Angular application recently which had to request for data from an API. Since I was using the Angular HttpClient, the response for the data request was wrapped in an observable by default.

Whenever a `GET` request for data was initiated, I wanted an animated ellipses to be displayed, indicating that the data retrieval process was ongoing. If the data was successfully retrieved, or an error occured during the retrieval process, the animated ellipses should exit the screen.

An observable is a data type that continually emits data to a subscriber that is attached to it. A subscriber is a data type that continually listens for data emitted from an observable it is subscribed to. When a subscriber subscribes to an observable, the subscriber is provided with three handlers to respond to the data that the observable emits. These three handlers are:

- The `next` handler - executed when the observable source emits a new value from its data sequence,
- The `error` handler - executed when an error occurs in the emission of values from the observable's data sequence, and
- The `complete` handler - executed when there is no more value available to be emitted from the observable sequence

Assuming the `getResults` method below returns an observable, the `next`, `error` and `complete` handlers are exemplified in its subscribe method as shown in the code snippet below

```ts
getResults().subscribe(
  results => console.log('Next handler executed with results: ', results),
  error => console.error('Error handler executed with error: ', error),
  () => console.log(`Complete handler executed. All values have been emitted`),
);
```

Being a newbie to observables, I placed the method that hid the animated ellipses in the `complete` method as demonstrated in the snippet below

```ts
getResults().subscribe(
  results => displayResults(results),
  error => notifyOnError(error.message),
  () => hideAnimatedEllipses(),
);
```

and the animated ellipses was hidden (as long as the request returned no errors). Whenever there was an error, the animated ellipses still danced around the user interface alongside the error message displayed.

In order to solve this, the first thing I did was to execute the `hideAnimatedEllipses()` method in the `complete` and `error` handlers. Sure thing! It worked, but then I got to know about the finalize operator, a better way to get the job done.

Learning about the finalize operator not only solved my problem, it also exposed the fault in my understanding of the three subscription handlers.

I got to find out that after the `error` handler is executed, further calls to the `next` handler will have no effect, and that after the `complete` handler is executed, further calls to `next` handler will have no effect too. That was why the animated ellipses continued to confidently dance on the user interface in the first instance even after the error message was displayed.

> **finalize**: The `finalize` operator is executed when an observable's source terminates on complete or on error.

I realised that in the execution of the `finalize` operator function is where the `hideAnimatedEllipses()` function should properly reside, and so the code was updated to the code snippet shown below

```ts
getResults()
  .pipe(finalize(() => hideAnimatedEllipses()))
  .subscribe(results => displayResults(results), error => notifyOnError(error.message));
```

In essense

> The **`next`** handler is executed when the observable source emits a new value from its data sequence,

> The **`error`** handler is executed when an error occurs in the emission of values from the observable's data sequence. After it is executed, further calls to `next` will have no effect, and

> The **`complete`** handler is executed when there is no more value available to be emitted from the observable sequence. After it is executed, further calls to `next` will have no effect

> The **`finalize`** operator is executed when an observable's source terminates on complete or on error.The code in the `finalize` operator is also executed when a subscriber unsubscribes from an observable, thus `finalize` acts like the `finally` method of a promise or a `try-catch-finally` block. Before RxJS v5.5, it was denoted as `finally`.

You can find more information about the `finalize` operator in the RxJS docs [here](http://reactivex.io/rxjs/file/es6/operators/finalize.js.html)

Cheers!😃
