# Flight of the Flux 3 - Hopping Threads and Schedulers

This blog post is the third in a series of posts that aim at providing a deeper look into [Reactor](https://github.com/reactor/reactor-core)’s more advanced concepts and inner workings.

In this post, we explore the threading model, how some (most) operators are concurrent agnostic, the `Scheduler` abstraction and how to hop from one thread to another mid-sequence with operators like `publishOn`.

This series is derived from the `Flight of the Flux` talk, which content I found to be more adapted to a blog post format.

The table below will be updated with links when the other posts are published, but here is the planned content:

1. [Assembly vs Subscription](https://spring.io/blog/2019/03/06/flight-of-the-flux-1-assembly-vs-subscription)
2. [Debugging caveats](https://spring.io/blog/2019/04/16/flight-of-the-flux-2-debugging-caveats)
3. Hopping Threads and Schedulers (this post)
4. Inner workings: work stealing
5. Inner workings: operator fusion

If you’re missing an introduction to *Reactive Streams* and the basic concepts of Reactor, head out to the site’s [learning section](https://projectreactor.io/learn) and the [reference guide](https://projectreactor.io/docs/core/release/reference).

Without further ado, let’s jump in:

## The Threading Model

Reactor operators generally are _concurrent agnostic_: they don't impose a particular threading model and just run on the `Thread` on which their `onNext` method was invoked.

As we saw in the first post of this series, the `Thread` that executes the subscription call also has an influence: the `subscribe` calls are chained until a data-producing `Publisher` is reached (the leftmost part of the chain of operators), then this `Publisher` offers a `Subscription` through `onSubscribe`, in turn passed down the chain, requested, etc... By default, again, this data production process starts on the `Thread` that initiated the subscription.

There is a general exception to this: operators that deal with a notion of **time**. Any such operator will default to running timers/delays/etc... on the `Schedulers.parallel()` scheduler.

A few other exceptions exist that also run on this `parallel()` `Scheduler`. They can be recognized by having at least one overload that takes a `Scheduler` parameter.

But what is a `Scheduler` and why do we need it?

## The `Scheduler` abstraction

In Reactor, a `Scheduler` is an abstraction that gives the user control about threading. A `Scheduler` can spawn `Worker` which are conceptually `Threads`, but are not necessarily backed by a `Thread` (we'll see an example of that later). A `Scheduler` also includes the notion of a **clock**, whereas the `Worker` is purely about scheduling tasks.

```java
interface Scheduler extends Disposable {
    
  Disposable schedule(Runnable task);
  Disposable schedule(Runnable task, long initialDelay, TimeUnit delayUnit);
  Disposable schedulePeriodically(Runnable task, long initialDelay, long perido, TimeUnit unit);
  
  long now(TimeUnit unit);
  
  Worker createWorker();
  
  interface Worker extends Disposable {
    Disposable schedule(Runnable task);
    Disposable schedule(Runnable task, long initialDelay, TimeUnit delayUnit);
    Disposable schedulePeriodically(Runnable task, long initialDelay, long perido, TimeUnit unit);
  }
}
```

Reactor comes with several default `Scheduler` implementations, each with its own specificity about how it manages `Workers`. They can be instantiated via the `Schedulers` factory methods. Here are rule of thumbs for their typical usage:

- `Schedulers.immediate()` can be used as a _null object_ for when an API requires a `Scheduler` but you don't want to change threads
- `Schedulers.single()` is for one-off tasks that can be run on a unique `ExecutorService`
- `Schedulers.parallel()` is good for CPU-intensive but short-lived tasks. It can execute `N` such tasks in parallel (by default `N == number of CPUs`)
- `Schedulers.elastic()` and `Schedulers.boundedElastic()` are good for more long-lived tasks (eg. blocking IO tasks). The `elastic` one spawns threads on-demand without a limit while the recently introduced `boundedElastic` does the same with a ceiling on the number of created threads.

Each flavor of `Scheduler` has a default global instance returned by the above methods, but one can create new instances using the `Schedulers.new***` factory methods (eg. `Schedulers.newParallel("myParallel", 10))` to create a custom parallel `Scheduler` where `N` = `10`).

The `parallel` flavor is backed by `N` workers each based on a `ScheduledExecutorService`. If you submit `N` long lived tasks to it, no more work can be executed, hence the affinity for _short-lived_ tasks.

The `elastic` flavor is also backed by workers based on `ScheduledExecutorService`, except it creates these workers on demand and pools them. A `Worker` that is no longer in used is returned to the pool on `dispose()` and will be kept here for the configured TTL duration, so new incoming tasks may reuse idle workers. However, it keeps on creating new workers if no idle `Worker` is available.

The `boundedElastic` flavor is very similar in concept to the `elastic` one except it places an upper bound to the number of `ScheduledExecutorService`-backed `Worker` it creates. Past this point, its `createWorker()` method returns a facade `Worker` that will enqueue tasks instead of submitting them immediately. As soon as a concrete `Worker` becomes available, it is swapped with the facade and starts actually submitting tasks (making it act like you only just submitted the task, including delayed ones). Additionally, one can put a cap on the total number of deferred tasks which can be enqueued by all the facade workers of the `Scheduler` instance.

### Are Schedulers Always Backed by an ExecutorService?

As we said above, no. We already saw an example actually: the `immediate() Scheduler`. This one doesn't modify which `Thread` the code is running on.

But there is a more useful example in the `reactor-test` library: the `VirtualTimeScheduler`. This `Scheduler` executes on the current `Thread`, but stamps all tasks submitted to it with the time at which they are supposed to be run.

It then manages a **virtual clock** (thanks to the fact that `Scheduler` also has the responsabilities of a clock) which can be manually advanced. When doing so, tasks that were queued to execute before or at the new virtual timestamp will be executed.

This is very useful in test scenarios where you have a `Flux` or `Mono` with long intervals/delays and you want to test the logic rather than the timing. For instance something like a `Mono.delay(Duration.ofHours(4))` can be run in under `100ms`...

One could also imagine implementing a `Scheduler` around a Actor system, the `ForkJoinPool`, upcoming Loom fibers, etc...

> **About the _main_ `Thread`**
>
> Often, people ask about switching back and forth between a `Scheduler`'s thread and the _main_ thread. Going from the main to a scheduler is obviously possible, **but going from an arbitrary thread to the _main_ thread is not possible**. That is plainly a Java limitation, as there is no way to submit tasks to the _main_ thread (e.g. there's no MainThreadExecutorService).

## Applying Schedulers to Operators

No that we're familiar with the building blocks of threading in Reactor, let's see how this translates in the world of operators.

We've already established that most operator continue their work on the `Thread` from which they were signalled, except for time-based operators (like `Mono.delay`, `bufferTimeout()`, etc...).

The philosophy of Reactor is to give you tools to do the right thing, by way of composing operators. Threading is not an exception: meet `subscribeOn` and `publishOn`.

These two operators simply take a `Scheduler` and will switch execution on one of that scheduler's `Worker`. There is of course a major difference between the two :)

### The `publishOn(Scheduler s)` operator

This is the basic operator you need when you want to hop threads.  Incoming signals from its source are _published_ on the given `Scheduler`, effectively switching threads to one of that scheduler's workers.

This is valid for the `onNext`, `onComplete` and `onError` signals. That is, signals that flow from an upstream source to a downstream subscriber.

So in essence, every processing step that appears below this operator will execute on the new `Scheduler` `s`, until another operator switches again (eg. another `publishOn`).

Let's take a deliberately sketchy example with blocking calls But remember, blocking calls in a reactive chain are always sketchy! :)

```java
Flux.fromIterable(firstListOfUrls) //contains A, B and C
    .map(url -> blockingWebClient.get(url))
    .subscribe(body -> System.out.println(Thread.currentThread().getName + " from first list, got " + body));

Flux.fromIterable(secondListOfUrls) //contains D and E
    .map(url -> blockingWebClient.get(url))
    .subscribe(body -> System.out.prinln(Thread.currentThread().getName + " from second list, got " + body));
```

In the above example, assuming this code is executed on the _main_ thread, each `Flux.fromIterable` emits the content of its `List` on that same `Thread`. We then use an imperative blocking web client inside a `map` to fetch the body of each `url`, which "inherits" that thread (and thus blocks it). The data-consuming lambda in each `subscribe` is thus also running on the main thread.

As a consequence, all these urls are processed sequentially on the main thread:

```
main from first list, got A
main from first list, got B
main from first list, got C
main from second list, got D
main from second list, got E
```

If we introduce `publishOn`, we can make this code more performant, so that the `Flux` don't block each other:

```java
Flux.fromIterable(firstListOfUrls) //contains A, B and C
    .publishOn(Schedulers.boundedElastic())
    .map(url -> blockingWebClient.get(url))
    .subscribe(body -> System.out.println(Thread.currentThread().getName + " from first list, got " + body));

Flux.fromIterable(secondListOfUrls) //contains D and E
    .publishOn(Schedulers.boundedElastic())
    .map(url -> blockingWebClient.get(url))
    .subscribe(body -> System.out.prinln(Thread.currentThread().getName + " from second list, got " + body));
```

Which could give us something like the following output:

```
boundedElastic-1 from first list, got A
boundedElastic-2 from second list, got D
boundedElastic-1 from first list, got B
boundedElastic-2 from second list, got E
boundedElastic-1 from first list, got C
```

First list and second list are interleaved now, great !

### The `subscribeOn(Scheduler s)` operator

In the preceding example we saw how `publishOn` could be used to offset blocking work on a separate Thread, by switching the publication of the _triggers_ for that blocking work (the urls to fetch) on a provided `Scheduler`.

Since the `map` operator runs on its source thread, switching that source thread by putting a `publishOn` before the `map` works as intended.

But what if that url-fetching method was written by somebody else, and they regrettably forgot to add the `publishOn`? Is there a way to influence the `Thread` **upstream**?

In a way, there is. That's where `subscribeOn` can come in handy.

This operator changes where the `subscribe` method is executed. And since the subscribe signal flows upward, it directly influences where the source `Flux` subscribes and starts generating data.

As a consequence, it can seem to act on the parts of the reactive chain of operators upward and downward (as long as there is no `publishOn` thrown in the mix):

```java
//code provided in library you have no write access to
final Flux<String> fetchUrls(List<String> urls) {
  return Flux.fromIterable(urls)
    .map(url -> blockingWebClient.get(url)); //oops!
}

//your code:
fetchUrls(A, B, C)
  .subscribeOn(Schedulers.boundedElastic())
  .subscribe(body -> System.out.println(Thread.currentThread().getName + " from first list, got " + body));

fetchUrls(D, E)
  .subscribeOn(Schedulers.boundedElastic())
  .subscribe(body -> System.out.prinln(Thread.currentThread().getName + " from second list, got " + body));
```

Like in our second `publishOn` example, that code will correctly output something like:

```
boundedElastic-1 from first list, got A
boundedElastic-2 from second list, got D
boundedElastic-1 from first list, got B
boundedElastic-2 from second list, got E
boundedElastic-1 from first list, got C
```

So what happened?

The `subscribe` calls are still running on the _main_ thread, but they propagate a `subscribe` signal to their source, `subscribeOn`. In turn `subscribeOn` propagates that same signal to its own source from `fetchUrls`, **but on a _boundedElastic_ `Worker`**.

In the `Flux` sequence returned by `fetchUrls`, the map is subscribed on the boundedElastic worker thread, and so is the `range`. The `range` starts generating data, still on the boundedElastic worker thread.

This continues down the data path, each subscriber executing `onNext` on its source thread, the `boundedElastic` one.

At last, the lambdas configured in the `subscribe(...)` call are also executed on the `boundedElastic` thread.

> **Important**
>
> It is important to distinguish the _act_ of subscribing and the lambda passed to the `subscribe()` method. This method subscribes to its source `Flux`, but the lambda are executed at the end of processing, when the data has flown through all the steps (including steps that hop to another thread),.
>
> So the `Thread` on which the lambda is executed might be different from the subscription `Thread` , ie. the thread on which the `subscribe` method is called.

And if we were the author of the `fetchUrls` library, we could make the code even more performant by letting each fetch run on its own `Worker`, by leveraging `subscribeOn` in a slightly different way:

```java
final Flux<String> betterFetchUrls(List<String> urls) {
  return Flux.fromIterable(urls)
    .flatMap(url -> 
             //wrap the blocking call in a Mono
             Mono.fromCallable(() -> blockingWebClient.get(url))
             //ensure that Mono is subscribed in an boundedElastic Worker
             .subscribeOn(Schedulers.boundedElastic())
    ); //each individual URL fetch runs in its own thread!
}
```



### And What If I Mix the Two?

`subscribeOn` will act throughout the subscribe phase, from bottom to top, then on the data path until it encounters a `publishOn` (or a time based operator).

Let's consider the following example:

```java
Flux.just("hello")
    .doOnNext(v -> System.out.println("just " + Thread.currentThread().getName()))
    .publishOn(Scheduler.boundedElastic())
    .doOnNext(v -> System.out.println("publish " + Thread.currentThread().getName()))
    .delayElements(Duration.ofMillis(500))
    .subscribeOn(Schedulers.elastic())
    .subscribe(v -> System.out.println(v + " delayed " + Thread.currentThread().getName()));
```

This will print:

```
just elastic-1
publish boundedElastic-1
hello delayed parallel-1
```

We should unpack what happened step by step:

- Here `subscribe` is called on the _main_ thread, but subscription is rapidly switched to the `elastic` scheduler due to the `subscribeOn` immediately above.
- All the operators above it are also subscribed on `elastic`, from bottom to top.
- `just` emits its value on the `elastic` scheduler.
- the first `doOnNext` receives that value on the same thread and prints it out: `just elastic-1`
- then on the top to bottom data path, we encounter the `publishOn`: data from `doOnNext` is propagated downstream on the `boundedElastic` scheduler.
- the second `doOnNext` receives its data on `boundedElastic` and prints `publish bounderElastic-1` accordingly.
- `delayElements` is a time operator, so by default it publishes data on the `Schedulers.parallel()` scheduler.
- on the data path, `subscribeOn` does nothing but propagating signal on the same thread.
- on the data path, the lambda(s) passed to `subscribe(...)` are executed on the thread in which data signals are received, so the lambda prints `hello delayed parallel-1`

## Conclusion

In this article, we’ve learned about the `Scheduler` abstraction and how it enables advanced usage like the `VirtualTimeScheduler`.

We then have learned how to switch threads (or rather `Scheduler` workers) in the middle of a reactive sequence, and what is the difference between `publishOn` and `subscribeOn`.

In the next instalment, we’ll dig deeper in the innards of the library to describe some optimizations that are in place to ensure Reactor's performance.

In the meantime, happy reactive coding!
