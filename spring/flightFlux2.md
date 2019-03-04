# Flight of the Flux 2 - Debugging Caveats

This blog post is the second in a series of posts that aim at providing a deeper look into [Reactor](https://github.com/reactor/reactor-core)'s more advanced concepts and inner workings.

It is derived from my `Flight of the Flux` talk, which content I found to be more adapted to a blog post format.

I'll update the table below with links when the other posts are published, but here is the planned content:

1. [Assembly vs Subscription](https://spring.io/blog/3570-flight-of-the-flux-1-assembly-vs-subscription)
2. Debugging caveats (this post)
3. Concurrent Agnostic
4. Schedulers and `publishOn` vs `subscribeOn`
5. Inner workings: work stealing
6. Inner workings: operator fusion

If you're missing an introduction to _Reactive Streams_ and the basic concepts of Reactor, head out to the site's [learning section](https://projectreactor.io/learn) and the [reference guide](https://projectreactor.io/docs/core/release/reference).

Without further ado, let's jump in:

## Debugging in a Reactive World

Switching from an imperative, blocking paradigm to a reactive, non-blocking one brings benefits but also comes with some caveats. One of these is the debugging experience. Why is that?

Primarily because you've learned to rely on the good old *stacktrace*, but suddenly this invaluable tool becomes far less valuable due to the **asynchronous** aspect of reactive programming. This is not specific to reactive programming though: as soon as you introduce asynchronous code, you create a boundary in the program between the code that _schedules_ and the code that _asynchronously executes_.

### Demonstrating the issue with vanilla async code

Let's take an example with an `ExecutorService` and a `Future` (no Reactor code here):

```java
	private static void imperative() throws ExecutionException, InterruptedException {
		final ScheduledExecutorService executor =
				Executors.newSingleThreadScheduledExecutor();

		int seconds = LocalTime.now().getSecond();
		List<Integer> source;
		if (seconds % 2 == 0) {
			source = IntStream.range(1, 11).boxed().collect(Collectors.toList());
		}
		else if (seconds % 3 == 0) {
			source = IntStream.range(0, 4).boxed().collect(Collectors.toList());
		}
		else {
			source = Arrays.asList(1, 2, 3, 4);
		}

		executor.submit(() -> source.get(5))  //line 76
		        .get();
	}
```

This example is a bit contrived, but let's imagine that we have these two out of three path in the code that can lead to the asynchronous task throwing an `IndexOutOfBoundsException`... How helpful would the stacktrace be?

```
java.util.concurrent.ExecutionException: java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 4
	at java.base/java.util.concurrent.FutureTask.report(FutureTask.java:122)
	at java.base/java.util.concurrent.FutureTask.get(FutureTask.java:191)
	at Scratch.imperative(Scratch.java:77)
	at Scratch.main(Scratch.java:50)
Caused by: java.lang.ArrayIndexOutOfBoundsException: Index 5 out of bounds for length 4
	at java.base/java.util.Arrays$ArrayList.get(Arrays.java:4351)
	at Scratch.lambda$imperative$0(Scratch.java:76)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:834)

```

We see that:

- the `Future`'s `get()` method threw an `ExecutionException`
- the cause is an `IndexOutOfBoundsException`
- the code throwing is in the `submit(() -> source.get(5))` **lambda** line 76
- it executed in a `FutureTask`, from something called a `ThreadPoolExecutor`, itself running in a `Thread`...
- we have two potential sources that could cause this but no idea which one is the culprit (which path was taken in the test prior to calling `submit()`).

Not terribly useful :-(

### Demonstrating the issue in Reactor

If we look for a Reactor equivalent to the above code, we can come up with this:

```java
	private static void reactive() {
		int seconds = LocalTime.now().getSecond();
		Mono<Integer> source;
		if (seconds % 2 == 0) {
			source = Flux.range(1, 10)
			             .elementAt(5);
		}
		else if (seconds % 3 == 0) {
			source = Flux.range(0, 4)
			             .elementAt(5);
		}
		else {
			source = Flux.just(1, 2, 3, 4)
			             .elementAt(5);
		}

		source.subscribeOn(Schedulers.parallel())
		      .block(); //line 97
	}
```

Which triggers the following stacktrace:

```
java.lang.IndexOutOfBoundsException
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.onComplete(MonoElementAt.java:153)
	at reactor.core.publisher.FluxArray$ArraySubscription.fastPath(FluxArray.java:176)
	at reactor.core.publisher.FluxArray$ArraySubscription.request(FluxArray.java:96)
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.request(MonoElementAt.java:92)
	at reactor.core.publisher.MonoSubscribeOn$SubscribeOnSubscriber.trySchedule(MonoSubscribeOn.java:186)
	at reactor.core.publisher.MonoSubscribeOn$SubscribeOnSubscriber.onSubscribe(MonoSubscribeOn.java:131)
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.onSubscribe(MonoElementAt.java:107)
	at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:53)
	at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:59)
	at reactor.core.publisher.MonoElementAt.subscribe(MonoElementAt.java:59)
	at reactor.core.publisher.Mono.subscribe(Mono.java:3711)
	at reactor.core.publisher.MonoSubscribeOn$SubscribeOnSubscriber.run(MonoSubscribeOn.java:123)
	at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:84)
	at reactor.core.scheduler.WorkerTask.call(WorkerTask.java:37)
	at java.base/java.util.concurrent.FutureTask.run(FutureTask.java:264)
	at java.base/java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:304)
	at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1128)
	at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:628)
	at java.base/java.lang.Thread.run(Thread.java:834)
	Suppressed: java.lang.Exception: #block terminated with an error
		at reactor.core.publisher.BlockingSingleSubscriber.blockingGet(BlockingSingleSubscriber.java:93)
		at reactor.core.publisher.Mono.block(Mono.java:1495)
		at Scratch.reactive(Scratch.java:97)
		at Scratch.main(Scratch.java:51)
```

- We see the `ArrayIndexOutOfBoundsException` again, hinting at a source that was too short for the `MonoElementAt` operator
- We see that it came from an `onComplete`, itself triggered by `request`... and a bunch of other steps in `reactor.core.publisher`
- With a bit of familiarity with these reactor methods, we _might_ deduce that the pipeline was made up of `range` (`FluxRange.subscribe`), `elementAt` and `subscribeOn`...
- It seems the throwing code was executed from the worker `Thread` of a `ThreadPoolExecutor`
- The trail goes cold here...

Worse, even if we did get rid of `subscribeOn` we'd still wouldn't discover which of the two possible error paths was triggered:

```java
	private static void reactiveNoSubscribeOn() {
		int seconds = LocalTime.now().getSecond();
		Mono<Integer> source;
		if (seconds % 2 == 0) {
			source = Flux.range(1, 10)
			             .elementAt(5);
		}
		else if (seconds % 3 == 0) {
			source = Flux.range(0, 4)
			             .elementAt(5);
		}
		else {
			source = Flux.just(1, 2, 3, 4)
			             .elementAt(5);
		}

		source.block(); //line 116
	}
```

Gives the stacktrace:

```
java.lang.IndexOutOfBoundsException
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.onComplete(MonoElementAt.java:153)
	at reactor.core.publisher.FluxArray$ArraySubscription.fastPath(FluxArray.java:176)
	at reactor.core.publisher.FluxArray$ArraySubscription.request(FluxArray.java:96)
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.request(MonoElementAt.java:92)
	at reactor.core.publisher.BlockingSingleSubscriber.onSubscribe(BlockingSingleSubscriber.java:49)
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.onSubscribe(MonoElementAt.java:107)
	at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:53)
	at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:59)
	at reactor.core.publisher.MonoElementAt.subscribe(MonoElementAt.java:59)
	at reactor.core.publisher.Mono.block(Mono.java:1494)
	at Scratch.reactiveNoSubscribeOn(Scratch.java:116)
	at Scratch.main(Scratch.java:52)
	Suppressed: java.lang.Exception: #block terminated with an error
		at reactor.core.publisher.BlockingSingleSubscriber.blockingGet(BlockingSingleSubscriber.java:93)
		at reactor.core.publisher.Mono.block(Mono.java:1495)
		... 2 more
```

That is because, as we saw previously, there is an additional "boundary" in code between _assembly_ and _subscription_. The trail only goes back to the point of **subscription** (here the `block()`) :-(

So using stacktraces for analysis and debugging purposes is harder in an asynchronous world, and even a bit harder in Reactor (because it is asynchronous and has a lazy-by-default approach with assembly vs subscription). But fortunately there are tools in the library to try and alleviate that fact.

## Making Things Better

### Back to classics: `log`

Remember when you sprinkled your imperative code with `print` statements? It might not be as cool as firing up the step debugger, but sometimes it is the quick and dirty solution you need.

In Reactor, you have the `log()` operator:

- It logs Reactive Stream signals: `onNext`, `onComplete`, `onError` (and **even** `onSubscribe`, `cancel` and `request`!)
- You can tune it to whitelist only part of these signals
- You can choose a particular `Logger` as well

In short, `log` is the quick and dirty solution to get an easy bird eye's view of what is going on at one step of your sequence. Use it liberally during development, with the possibility of specifying a "name" to each `log` call to differentiate them.

Using `log(String)` can be diverted to get a hint at which source causes the error:

```java
	private static void log() {
		int seconds = LocalTime.now().getSecond();
		Mono<Integer> source;
		if (seconds % 2 == 0) {
			source = Flux.range(1, 10)
			             .elementAt(5)
			             .log("source A");
		}
		else if (seconds % 3 == 0) {
			source = Flux.range(0, 4)
			             .elementAt(5)
			             .log("source B");
		}
		else {
			source = Flux.just(1, 2, 3, 4)
			             .elementAt(5)
			             .log("source C");
		}

		source.block(); //line 138
	}
```

The stacktrace itself isn't much more interesting (apart from mentioning the `MonoLogFuseable` class, but the log itself contains this interesting tidbit:

```
17:01:23.711 [main] INFO  source C - | onSubscribe([Fuseable] MonoElementAt.ElementAtSubscriber)
17:01:23.716 [main] INFO  source C - | request(unbounded)
17:01:23.717 [main] ERROR source C - | onError(java.lang.IndexOutOfBoundsException)
17:01:23.721 [main] ERROR source C - 
java.lang.IndexOutOfBoundsException: null
```

At least we get our hardcoded `source C` label...

### Enriching stacktraces with Debug Mode

Another approach that is available in Reactor is to try and get back the assembly information in the runtime stacktraces.

This can be done by activating the so-called "debug mode" via the `Hooks` class:

```java
Hooks.onOperatorDebug();
```

What does it do? It makes each operator instantiation (aka assembly) capture a stacktrace and keep it for later.

If an `onError` reaches one operator, it will attach that assembly stacktrace to the `onError` 's `Throwable` (as a **suppressed `Exception`**). As a result, when you see the stacktrace you'll get a more complete picture of both the runtime AND the assembly.

With debug mode on, in our earlier example we would be able to see which assembly path was taken and which source was actually processed:

```java
	private static void hook() {
		Hooks.onOperatorDebug();
		try {
			int seconds = LocalTime.now().getSecond();
			Mono<Integer> source;
			if (seconds % 2 == 0) {
				source = Flux.range(1, 10)
				             .elementAt(5); //line 149
			}
			else if (seconds % 3 == 0) {
				source = Flux.range(0, 4)
				             .elementAt(5); //line 153
			}
			else {
				source = Flux.just(1, 2, 3, 4)
				             .elementAt(5); //line 157
			}

			source.block(); //line 160
		}
		finally {
			Hooks.resetOnOperatorDebug();
		}
	}
```

Which produces the following stacktrace:

```java
java.lang.IndexOutOfBoundsException
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.onComplete(MonoElementAt.java:153)
(...)
	at reactor.core.publisher.Mono.block(Mono.java:1494)
	at Scratch.hook(Scratch.java:160)
	at Scratch.main(Scratch.java:54)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly trace from producer [reactor.core.publisher.MonoElementAt] :
	reactor.core.publisher.Flux.elementAt(Flux.java:4367)
	Scratch.hook(Scratch.java:157)
Error has been observed by the following operator(s):
	|_	Flux.elementAt ⇢ Scratch.hook(Scratch.java:157)
```

Notice the last line? Yay :-D

### Bringing the cost down with `checkpoint`

One drawback of using `Hooks.onOperatorDebug()` is that it does the assembly stacktrace capture **for every single operator used in the application**. Filling a single stacktrace is a costly operation, so it goes without saying that this can have an heavy impact on performance. As a result, this is only recommended in a development setting.

Fortunately, you can bring the cost down a little if you identify parts of your codebase that are prone to that sort of source ambiguity.

By using the `checkpoint()` operator, it is possible to activate the assembly trace capture only at that specific point in the codebase. You can even do entirely without the filling of a stacktrace if you give the checkpoint a unique and meaningful name using `checkpoint(String)`:

```java
	private static void checkpoint() {
		int seconds = LocalTime.now().getSecond();
		Mono<Integer> source;
		if (seconds % 2 == 0) {
			source = Flux.range(1, 10)
			             .elementAt(5)
			             .checkpoint("source range(1,10)");
		}
		else if (seconds % 3 == 0) {
			source = Flux.range(0, 4)
			             .elementAt(5)
			             .checkpoint("source range(0,4)");
		}
		else {
			source = Flux.just(1, 2, 3, 4)
			             .elementAt(5)
			             .checkpoint("source just(1,2,3,4)");
		}

		source.block(); //line 186
	}
```

This produces the following stacktrace:

```
java.lang.IndexOutOfBoundsException
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.onComplete(MonoElementAt.java:153)
	at reactor.core.publisher.FluxArray$ArraySubscription.fastPath(FluxArray.java:176)
	at reactor.core.publisher.FluxArray$ArraySubscription.request(FluxArray.java:96)
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.request(MonoElementAt.java:92)
	at reactor.core.publisher.FluxOnAssembly$OnAssemblySubscriber.request(FluxOnAssembly.java:438)
	at reactor.core.publisher.BlockingSingleSubscriber.onSubscribe(BlockingSingleSubscriber.java:49)
	at reactor.core.publisher.FluxOnAssembly$OnAssemblySubscriber.onSubscribe(FluxOnAssembly.java:422)
	at reactor.core.publisher.MonoElementAt$ElementAtSubscriber.onSubscribe(MonoElementAt.java:107)
	at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:53)
	at reactor.core.publisher.FluxArray.subscribe(FluxArray.java:59)
	at reactor.core.publisher.MonoElementAt.subscribe(MonoElementAt.java:59)
	at reactor.core.publisher.MonoOnAssembly.subscribe(MonoOnAssembly.java:61)
	at reactor.core.publisher.Mono.block(Mono.java:1494)
	at Scratch.checkpoint(Scratch.java:186)
	at Scratch.main(Scratch.java:55)
	Suppressed: reactor.core.publisher.FluxOnAssembly$OnAssemblyException: 
Assembly site of producer [reactor.core.publisher.MonoElementAt] is identified by light checkpoint [source just(1,2,3,4)].
```

Notice the `is identified by light checkpoint [source just(1,2,3,4)].`, which gives us our culprit (because we used a meaningful description for the checkpoint).

## Conclusion

In this article, we've learned that stacktraces can be less useful in asynchronous programming. This effect is further compounded by the lazy way Reactor let you build reactive sequences.

We've looked at the worst cases that can be encountered and at several ways this problem can be lessened.

The whole code can be found in a gist [here](https://gist.github.com/simonbasle/a4c6cfe79071610e1fe3bd27984d4381).

In the next instalment, we'll see why we sometimes say that Reactor is "Concurrent Agnostic".

In the meantime, happy reactive coding!
