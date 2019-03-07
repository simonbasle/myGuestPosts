# Flight of the Flux 1 - Assembly vs Subscription

This blog post is the first in a series of posts that aim at providing a deeper look into [Reactor](https://github.com/reactor/reactor-core)'s more advanced concepts and inner workings.

It is derived from my `Flight of the Flux` talk, which content I found to be more adapted to a blog post format.

I'll update the table below with links when the other posts are published, but here is the planned content:

1. Assembly vs Subscription (this post)
2. Debugging caveats
3. Concurrent Agnostic
4. Schedulers and `publishOn` vs `subscribeOn`
5. Inner workings: work stealing
6. Inner workings: operator fusion

If you're missing an introduction to _Reactive Streams_ and the basic concepts of Reactor, head out to the site's [learning section](https://projectreactor.io/learn) and the [reference guide](https://projectreactor.io/docs/core/release/reference).

Without further ado, let's jump in:

## Assembly Time

When you first learn about _Reactive Streams_ and _reactive programming_ on the JVM, the first thing you learn is the high-level relationship between `Publisher` and `Subscriber`: one produces data, the other consumes it. Simple right? Furthermore, it appears that the `Publisher` **pushes** data to the `Subscriber`.

But when working with Reactive Streams libraries like Reactor (or RxJava2), you quickly come across the following mantra:

> Nothing Happens Until You Subscribe

Sometimes, you might read that both libraries implement a "push-pull hybrid model". Hang on a minute! **pull**?

We'll get back to it, but to understand that sentence you first need to realize that, by default, Reactor's reactive types are _lazy_.

Calling methods on a `Flux` or `Mono` (the _operators_) doesn't immediately trigger the behavior. Instead, a new instance of `Flux` (or `Mono`) is returned, on which you can continue composing further operators. You thus create a _chain of operators_ (or an operator acyclic graph), which represents your **asynchronous processing pipeline**.

This **declarative** phase is called **assembly time**.

Let's take an example where a client side application makes an HTTP request to a server, expecting an HttpResponse:

```java
Mono<HttpResponse> httpSource = makeHttpRequest();
Mono<Json> jsonSource = httpSource.map(req -> parseJson(req));
Mono<String> quote = jsonSource.map(json -> json.getString("quote"));
//at this point, no HTTP request has been made
```

This can be simplified using the fluent API:

```java
Mono<String> quote = makeHttpRequest()
    .map(req -> parseJson(req))
    .map(json -> json.getString("quote"));
```

Once you are done declaring your pipeline, there are two situations: either you pass the `Flux`/`Mono` representing the processing pipeline down to another piece of code or you trigger the pipeline.

The former means that the code to which you return the `Mono` might apply other operators, resulting in a derived new pipeline. Since the operators create new instances (it's like an onion), your own `Mono` is not mutated, so it could be further decorated several times with widely different results:

```java
//you could derive a `Mono<String>` of odd-length strings vs even-length ones
Mono<String> evenLength = quote.filter(str -> str.length() % 2 == 0);
Mono<String> oddLength = quote.filter(str -> str.length() % 2 == 1);

//or even a `Flux<String>` of words in a quote
Flux<String> words = quote.flatMapMany(quote -> Flux.fromArray(quote.split(" ")));

//by this point, none of the 3 "pipelines" have triggered an HTTP request
```
Compare that with a `CompletableFuture`, which is not lazy in nature: once you have a reference to the `CompletableFuture`, it means the processing is already ongoing...

With that in mind, let's look into how to trigger the reactive pipeline.

## Subscription Time

So far, we've _assembled an asynchronous pipeline_. That is, we've instantiated `Flux` and `Mono` variables through the use of _operators_, that results other `Flux`/`Mono` with behavior layered like an onion.

But the data hasn't started flowing through each of these declared pipelines yet.

That's because the trigger for the data to flow is not the declaration of the pipeline, but rather the **subscription** to it. Remember:

> Nothing Happens Until You Subscribe

Subscribing is the act of saying "ok, this pipeline represent a transformation of data, and I'm interested in the final form of that data". The most common way of doing so is by calling `Flux.subscribe(valueConsumer, errorConsumer)`.

That signalling of interest is propagated backwards through the chain of operators, up until the _source_ operator, the `Publisher` that actually produces the initial data:

```java
makeHttpRequest() //<5>
    .map(req -> parseJson(req)) //<4>
    .map(json -> json.getString("quote")) //<3>
    .flatMapMany(quote -> Flux.fromArray(quote.split(" "))) //<2>
    .subscribe(System.out::println, Throwable::printStackTrace); //<1>
```

1. we subscribe to the words `Flux`, stating that we want to print each word to the console (and print the stack trace of any error)
2. that interest is signalled to the `flatMapMany` step...
3. ...which signals it up the chain to the json `map` step...
4. ...then the request `map` step...
5. ...to finally reach the `makeHttpRequest()` (which we'll consider our source)

At this point, the source is triggered. It generates the data in the appropriate way: here it would make an HTTP request to a JSON-producing endpoint and then emit the HTTP response.

From there on, we're in _execution time_. The data has started flowing through the pipeline (in the more natural top-to-bottom order, or _upstream_ to _downstream_):

1. The `HttpResponse` is emitted to the `parseJson` `map`
2. It extracts the JSON body and emits it to the `getString` `map` 
3. Which extracts the _quote_ and passes it to the `flatMapMany`
4. The `flatMapMany` splits the quote into words and emit each word individually
5. The value handler in the `subscribe` is notified of each word, printing these to the console, one per line

Hopefully that helps you understand the difference between assembly time and subscription/execution time!

## Cold vs Hot

Right after explaining the difference and introducing this mantra is probably a good time to introduce an exception :laughing: 

> Nothing happens until you subscribe... **until something does**

### Cold

So far, we've been dealing with a flavor of `Flux` and `Mono` sources called a **Cold `Publisher`**. As we've explained, these `Publishers` are lazy and only generate data when there is a `Subscription`. Furthermore, they generate the data anew for each individual `Subscription`.

In our example of an HTTP response `Mono`, the HTTP request would be performed for each subscription:  

```java
Mono<String> evenLength = quote.filter(str -> str.length() % 2 == 0);
Mono<String> oddLength = quote.filter(str -> str.length() % 2 == 1);
Flux<String> words = quote.flatMapMany(quote -> Flux.fromArray(quote.split(" ")));

evenLength.subscribe(); //this triggers an HTTP request
oddLength.subscribe(); //this triggers another HTTP request
words.subscribe(); //this triggers a third HTTP request
```

On a side note, some operators' behavior imply multiple subscriptions. For example `retry` re-subscribe to its source in case of an error (`onError` signal), while `repeat` does the same for the `onComplete` signal.

So for a cold source like the HTTP request, something like `retry` would re-perform the request thus allowing to recover from a transient server-side error, for instance.

### Hot

A **Hot `Publisher`** on the other hand isn't as clear-cut: it doesn't necessarily need a `Subscriber` to start pumping data. It doesn't necessarily re-generate dedicated data per each new `Subscriber` either.

To illustrate that, let's introduce a new cold publisher example, then we'll show how to turn that _cold_ publisher into a _hot_ one:



```java
Flux<Long> clockTicks = Flux.interval(Duration.ofSeconds(1));

clockTicks.subscribe(tick -> System.out.println("clock1 " + tick + "s");

Thread.sleep(2000);

clockTicks.subscribe(tick -> System.out.println("\tclock2 " + tick + "s");
```

This prints:

```
clock1 1s
clock1 2s
clock1 3s
    clock2 1s
clock1 4s
    clock2 2s
clock1 5s
    clock2 3s
clock1 6s
    clock2 4s
```

We can turn the `clockTicks` source into a hot one by invoking `share()`:

```java
Flux<Long> coldTicks = Flux.interval(Duration.ofSeconds(1));
Flux<Long> clockTicks = coldTicks.share();

clockTicks.subscribe(tick -> System.out.println("clock1 " + tick + "s");

Thread.sleep(2000);

clockTicks.subscribe(tick -> System.out.println("\tclock2 " + tick + "s");
```

It yields the following result instead:

```
clock1 1s
clock1 2s
clock1 3s
    clock2 3s
clock1 4s
    clock2 4s
clock1 5s
    clock2 5s
clock1 6s
    clock2 6s

```

You see that the two subscriptions now share the same ticks of the clock. `share()` converts cold to hot by letting the source multicast elements to new `Subscribers`, but **only the elements that are emitted after these new subscriptions**. Since `clock2` has subscribed 2 seconds later, it missed early emissions `1s` and `2s`.

So hot publishers can be less lazy, even though they generally require at least an initial `Subscription` to trigger data flow.

## Conclusion

In this article, we've learned about the difference between instantiating a `Flux` / chaining operator (aka **Assembly time**), triggering it (aka **Subscription time**)  and executing it (aka **Execution time**).

We've thus learned that `Flux` and `Mono` are mostly lazy (aka **cold `Publisher`**): **nothing happens until you subscribe** to them.

Finally, we've learned about an alternative flavor of `Flux` and `Mono`, dubbed the **hot `Publisher`**, which behaves a little differently and is less lazy.

In the next instalment, we'll see why these three phases make a major difference in how you as a developer would debug reactor-based code.

In the meantime, happy reactive coding!
