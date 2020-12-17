---
layout: post
current: post
cover: assets/images/2019-03-24-suspend-modifier.png
navigation: True
title: Simplifying APIs with coroutines and Flow
date: 2020-12-16 00:00:00
tags: [coroutines]
class: post-template
subclass: 'post'
author: manuel
---

Learn how to create your own coroutine adapters and see how they work under the hood

If you’re a library author, you might want to make your Java-based or callback-based libraries easier to consume from Kotlin using coroutines and Flow. Alternatively, if you’re an API consumer, you may be willing to adapt a 3rd party API surface to coroutines to make them more Kotlin friendly.

This article covers how to simplify APIs using coroutines and Flow as well as how to build your own adapter using `suspendCancellableCoroutine` and `callbackFlow` APIs. For the most curious ones, those APIs will be dissected and you’ll see how they work under the hood.

If you prefer to watch a video about this topic, check this one out:

<iframe width="560" height="315" src="https://www.youtube.com/embed/OmHePYcHbyQ" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Check existing coroutine adapters

Before writing your own wrappers for existing APIs, check if an adapter or [extension function](https://medium.com/androiddevelopers/extend-your-code-readability-with-kotlin-extensions-542bf702aa36) is available for your use case. There are existing libraries with coroutine adapters for common types.

### Future types

For future types, there are integrations for Java 8’s [CompletableFuture](https://github.com/Kotlin/kotlinx.coroutines/blob/master/integration/kotlinx-coroutines-jdk8/src/future/Future.kt), and Guava’s [ListenableFuture](https://github.com/Kotlin/kotlinx.coroutines/blob/master/integration/kotlinx-coroutines-guava/src/ListenableFuture.kt). This is not an exhaustive list, search online if an adapter for your future type already exists.

```kotlin
// Awaits completion of CompletionStage without blocking a thread
suspend fun <T> CompletionStage<T>.await(): T 

// Awaits completion of ListenableFuture without blocking a thread
suspend fun <T> ListenableFuture<T>.await(): T
```

With these functions, you can get rid of callbacks and just suspend the coroutine until the future result comes back.

### Reactive Streams

For reactive stream libraries, there are integrations for [RxJava](https://github.com/Kotlin/kotlinx.coroutines/tree/master/reactive/kotlinx-coroutines-rx3), [Java 9 APIs](https://github.com/Kotlin/kotlinx.coroutines/tree/master/reactive/kotlinx-coroutines-jdk9), and [reactive streams](https://github.com/Kotlin/kotlinx.coroutines/tree/master/reactive/kotlinx-coroutines-reactive) libraries.

```kotlin
// Transforms the given reactive Publisher into Flow.
fun <T : Any> Publisher<T>.asFlow(): Flow<T>
```

These functions convert a reactive stream into Flow.

### Android specific APIs

For Jetpack libraries or Android platform APIs, take a look at the [Jetpack KTX libraries list](https://developer.android.com/kotlin/ktx/extensions-list). Currently, more than 20 libraries have a KTX version, creating sweet idiomatic versions of Java APIs, ranging from SharedPreferences to ViewModels, SQLite and even Play Core.

### Callbacks

Callbacks are a very common solution for asynchronous communication. In fact, we use them for the Java programming language solution in the [Running tasks in background thread guide](https://developer.android.com/guide/background/threading). However, they come with some drawbacks: this design leads to nested callbacks which ends up in incomprehensible code. Also, error handling is more complicated as there isn’t an easy way to propagate them.

In Kotlin, you can simplify calling callbacks using coroutines, but for that, you’ll need to build your own adapter.

## Build your own adapter

If you don’t find an adapter for your use case, it’s usually quite straightforward to write your own. **For one-shot async calls, use the [`suspendCancellableCoroutine`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/suspend-cancellable-coroutine.html) API. For streaming data, use the [`callbackFlow`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/callback-flow.html) API**.

As an exercise, the following examples will use the [Fused Location Provider](https://developers.google.com/android/reference/com/google/android/gms/location/FusedLocationProviderClient.html) API from Google Play Services to get location data. The API surface is simple but it uses callbacks to perform async operations. With coroutines, we can get rid of those callbacks that can quickly make our code unreadable when the logic gets complicated.

In case you want to explore other solutions, you can get inspiration from the source code of all the functions linked above.

### One-shot async calls

The [Fused Location Provider](https://developers.google.com/android/reference/com/google/android/gms/location/FusedLocationProviderClient.html) API provides the [`getLastLocation`](https://developers.google.com/android/reference/com/google/android/gms/location/FusedLocationProviderClient#getLastLocation()) method to obtain the [last known location](https://developer.android.com/training/location/retrieve-current). The ideal API for coroutines is a suspend function that returns exactly that.

> Note that this API returns a [`Task`](https://developers.google.com/android/reference/com/google/android/gms/tasks/Task) and there’s already an [adapter](https://github.com/Kotlin/kotlinx.coroutines/blob/master/integration/kotlinx-coroutines-play-services/src/Tasks.kt) available for it. However, for learning purposes, we’ll use it as an example.

We can have a better API by creating an extension function on `FusedLocationProviderClient`:

```kotlin
suspend fun FusedLocationProviderClient.awaitLastLocation(): Location
```

As this is a one-shot async operation, we use the `suspendCancellableCoroutine` function: a low-level building block for creating suspending functions from the coroutines library.

`suspendCancellableCoroutine` executes the block of code passed to it as a parameter, then suspends the coroutine execution while waiting for the signal to continue. The coroutine will resume executing when the `resume` or `resumeWithException` method is called in the coroutine’s [`Continuation`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/) object. For more information about continuations, check out the _[suspend modifier under the hood article](https://manuelvivo.dev/suspend-modifier)_.

We use the callbacks that can be added to the `getLastLocation` method to resume the coroutine appropriately. See the implementation below:

```kotlin
// Extension function on FusedLocationProviderClient, returns last known location
suspend fun FusedLocationProviderClient.awaitLastLocation(): Location =

  // Create a new coroutine that can be cancelled
  suspendCancellableCoroutine<Location> { continuation ->

    // Add listeners that will resume the execution of this coroutine
    lastLocation.addOnSuccessListener { location ->
      // Resume coroutine and return location
      continuation.resume(location)
    }.addOnFailureListener { e ->
      // Resume the coroutine by throwing an exception
      continuation.resumeWithException(e)
    }

    // End of the suspendCancellableCoroutine block. This suspends the
    // coroutine until one of the callbacks calls the continuation parameter.
  }
```

Note: Although you will also find a non-cancellable version of this coroutine builder in the coroutines library (i.e. [`suspendCoroutine`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/suspend-coroutine.html)), it is preferable to always choose [`suspendCancellableCoroutine`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/suspend-cancellable-coroutine.html) to handle cancellation of the coroutine scope, or to propagate cancellation from the underlying API.

#### suspendCancellableCoroutine under the hood

Internally, [`suspendCancellableCoroutine`](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/CancellableContinuation.kt#L305) uses [`suspendCoroutineUninterceptedOrReturn`](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/src/kotlin/coroutines/intrinsics/Intrinsics.kt#L41) to get the `Continuation` of the coroutine inside a suspend function. That `Continuation` object is intercepted by a [`CancellableContinuation`](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/CancellableContinuation.kt) that will control the lifecycle of that coroutine from that point (its [implementation](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/CancellableContinuationImpl.kt) has the functionality of a `Job` with some restrictions).

After that, the lambda passed to `suspendCancellableCoroutine` will be executed and the coroutine will either resume immediately if the lambda returns a result or will be suspended until the `CancellableContinuation` is resumed manually from the lambda.

See my own comments in the following code snippet (following the [original implementation](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/CancellableContinuation.kt#L305)) to understand what’s happening:

```kotlin
public suspend inline fun <T> suspendCancellableCoroutine(
  crossinline block: (CancellableContinuation<T>) -> Unit
): T =
  // Get the Continuation object of the coroutine that it's running this suspend function
  suspendCoroutineUninterceptedOrReturn { uCont ->

    // Take over the control of the coroutine. The Continuation's been
    // intercepted and it follows the CancellableContinuationImpl lifecycle now
    val cancellable = CancellableContinuationImpl(uCont.intercepted(), /* ... */)
    /* ... */
 
    // Call block of code with the cancellable continuation
    block(cancellable)
        
    // Either suspend the coroutine and wait for the Continuation to be resumed
    // manually in `block` or return a result if `block` has finished executing
    cancellable.getResult()
  }
```

To know more about how suspend functions work under the hood, check out the _[suspend modifier under the hood article](https://manuelvivo.dev/suspend-modifier)_.

### Streaming data

If instead we wanted to receive periodic location updates (using the [`requestLocationUpdates`](https://developers.google.com/android/reference/com/google/android/gms/location/FusedLocationProviderClient#requestLocationUpdates(com.google.android.gms.location.LocationRequest,%20com.google.android.gms.location.LocationCallback,%20android.os.Looper)) function) whenever the user’s device moves in the real world, we’d need to create a stream of data using [Flow](https://kotlinlang.org/docs/reference/coroutines/flow.html). The ideal API would look like this:

```kotlin
fun FusedLocationProviderClient.locationFlow(): Flow<Location>
```

To convert streaming callback-based APIs to Flow, use the [`callbackFlow`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/callback-flow.html) flow builder that creates a new flow. In the `callbackFlow` lambda, we’re in the context of a coroutine, therefore, suspend functions can be called. Unlike the [`flow`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/flow.html) flow builder, `channelFlow` allows values to be emitted from a different `CoroutineContext` or outside a coroutine, with the [`offer`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.channels/-send-channel/offer.html) method.

Normally, flow adapters using `callbackFlow` follow these three generic steps:

1. Create the callback that adds elements into the flow using `offer`.
2. Register the callback.
3. Wait for the consumer to cancel the coroutine and unregister the callback.

Applying this recipe to this use case, we get the following implementation:

```kotlin
// Send location updates to the consumer 
fun FusedLocationProviderClient.locationFlow() = callbackFlow<Location> {
  // A new Flow is created. This code executes in a coroutine!

  // 1. Create callback and add elements into the flow
  val callback = object : LocationCallback() {
    override fun onLocationResult(result: LocationResult?) {
      result ?: return // Ignore null responses
      for (location in result.locations) {
        try {
          offer(location) // Send location to the flow
        } catch (t: Throwable) {
          // Location couldn't be sent to the flow 
        }
      }
    }
  }

  // 2. Register the callback to get location updates by calling requestLocationUpdates
  requestLocationUpdates(
    createLocationRequest(),
    callback,
    Looper.getMainLooper()
  ).addOnFailureListener { e ->
    close(e) // in case of error, close the Flow
  }

  // 3. Wait for the consumer to cancel the coroutine and unregister
  // the callback. This suspends the coroutine until the Flow is closed.
  awaitClose {
    // Clean up code goes here
    removeLocationUpdates(callback)
  }
}
```

#### callbackFlow under the hood

Internally, `callbackFlow` uses a [channel](https://kotlinlang.org/docs/reference/coroutines/channels.html), which is conceptually very similar to a blocking [queue](https://en.wikipedia.org/wiki/Queue_(abstract_data_type)). A channel is configured with a `capacity`: the number of elements that can be buffered. The channel created in `callbackFlow` has the default capacity of 64 elements. When adding a new element to an already full channel, `send` will suspend the producer until there’s space for the new element in the channel whereas `offer` won’t add the element to the channel and will return `false` immediately.

#### awaitClose under the hood

Interestingly, `awaitClose` uses `suspendCancellableCoroutine` under the hood. See my own comments in the following code snippet (following the [original implementation](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/channels/Produce.kt#L49)) to understand what’s happening:

```kotlin
public suspend fun ProducerScope<*>.awaitClose(block: () -> Unit = {}) {
  /* ... */
  try {
    // Suspend the coroutine with a cancellable continuation
    suspendCancellableCoroutine<Unit> { cont ->
      // Suspend forever and resume the coroutine successfully only 
      // when the Flow/Channel is closed
      invokeOnClose { cont.resume(Unit) }
    }
  } finally {
    // Always execute caller's clean up code
    block()
  }
}
```

#### Reusing the Flow

Flows are cold and lazy unless specified otherwise with intermediate operators such as [`conflate`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/conflate.html). This means that the builder block will be executed each time a terminal operator is called on the flow. This might not be a huge problem in our case as adding new location listeners is cheap, however, it might make a difference in other implementations.

```kotlin
val FusedLocationProviderClient.locationFlow() = callbackFlow<Location> {
  /* ... */
}.shareIn(
  // Make the flow follow the applicationScope
  applicationScope,
  // Emit the last emitted element to new collectors
  replay = 1,
  // Keep the producer active while there are active subscribers
  started = SharingStarted.WhileSubscribed()
)
```

To learn more about best practices for adding an `applicationScope` to your app, check out this [article](https://manuelvivo.dev/coroutines-cancellation-exceptions-4).

---

Consider creating coroutine adapters to make your APIs or existing APIs concise, readable and Kotlin idiomatic. First check if the adapter is already available and if not, create your own using `suspendCancellableCoroutine` for one-shot calls and `callbackFlow` for streaming data.

To get hands-on this topic, check out the [_Building a Kotlin extensions library_ codelab](https://codelabs.developers.google.com/codelabs/building-kotlin-extensions-library).

