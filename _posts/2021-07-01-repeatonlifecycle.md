---
layout: post
current: post
cover: assets/images/2021-07-01-repeatonlifecycle.png
navigation: True
title: repeatOnLifecycle API design story
date: 2021-07-01 00:00:00
tags: [coroutines]
class: post-template
subclass: 'post'
author: manuel
---

Learn the design decisions behind the Lifecycle.repeatOnLifecycle API.

In this blog post, you’ll learn the design decisions behind the `Lifecycle.repeatOnLifecycle` API and why we removed some of the helper functions we added in the first alpha version of the 2.4.0 [`lifecycle-runtime-ktx`](https://developer.android.com/jetpack/androidx/releases/lifecycle) library.

Along the way, you’ll see how certain coroutines APIs can be dangerous to use in some scenarios, how difficult naming is, and why we decided to keep only the low-level suspend APIs in the library.

Also, you’ll realize all API decisions require some tradeoffs regarding complexity, readability, and how error prone the API is.

Special shout-out to [Adam Powell](https://twitter.com/adamwp), [Wojtek Kaliciński](https://twitter.com/wkalic), [Ian Lake](https://twitter.com/ianhlake), and [Yigit Boyar](https://twitter.com/yigitboyar) for giving feedback and discussing the shape of these APIs.

> Note: If you’re looking for `repeatOnLifecycle` guidance, check out the [A safer way to collect flows from Android UIs blog post](https://manuelvivo.dev/coroutines-addrepeatingjob).

## repeatOnLifecycle

The `Lifecycle.repeatOnLifecycle` API was primarily born to allow safer Flow collection from the UI layer in Android. Its restartable behavior, that takes into consideration the UI lifecycle, makes it the perfect default API to process items only when the UI is visible on the screen.

> Note: `LifecycleOwner.repeatOnLifecycle` is also available. It delegates the functionality to its Lifecycle. With this, any code that’s already part of a `LifecycleOwner` scope can omit the explicit receiver.

**`repeatOnLifecycle` is a suspend function**. As such, it needs to be executed within a coroutine. `repeatOnLifecycle` suspends the calling coroutine, and then runs a given suspend `block` that you pass as a parameter in a new coroutine each time the given lifecycle reaches a target state or higher. If the lifecycle state falls below the target, the coroutine launched for the `block` is cancelled. Lastly, the `repeatOnLifecycle` function itself won’t resume the calling coroutine until the lifecycle is `DESTROYED`.

Let’s see this API in action. If you read my previous [A safer way to collect flows from Android UIs blog post](https://manuelvivo.dev/coroutines-addrepeatingjob), none of this should come as a surprise to you.

```kotlin
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        // Create a new coroutine from the lifecycleScope
        // since repeatOnLifecycle is a suspend function
        lifecycleScope.launch {
            // Suspend the coroutine until the lifecycle is DESTROYED.
            // repeatOnLifecycle launches the block in a new coroutine every time the 
            // lifecycle is in the STARTED state (or above) and cancels it when it's STOPPED.
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                // Safely collect from locations when the lifecycle is STARTED
                // and stop collecting when the lifecycle is STOPPED
                someLocationProvider.locations.collect {
                    // New location! Update the map
                }
            }
            // Note: at this point, the lifecycle is DESTROYED!
        }
    }
}
```

> Note: if you’re interested in how `repeatOnLifecycle` is implemented, here’s a [link to its source code](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-runtime-ktx/src/main/java/androidx/lifecycle/RepeatOnLifecycle.kt;l=63).

## Why it’s a suspend function

**A suspend function is the best choice** for this restarting behavior as it preserves the calling context. It _respects_ the `Job` tree of the calling coroutine. As `repeatOnLifecycle`’s implementation uses `suspendCancellableCoroutine` under the hood, it cooperates with cancellation: cancelling the calling coroutine also cancels `repeatOnLifecycle` and its restarting `block`.

Also, we can add more APIs on top of `repeatOnLifecycle` such as the [`Flow.flowWithLifecycle`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-runtime-ktx/src/main/java/androidx/lifecycle/FlowExt.kt;l=87) flow operator. More importantly, it also allows you to create helper functions on top of this API if that’s what your project needs. That’s what we tried to do with the `LifecycleOwner.addRepeatingJob` API that we added in `lifecycle-runtime-ktx:2.4.0-alpha01` and, in fact, removed in `alpha02`.

## Removing the addRepeatingJob API

The `LifecycleOwner.addRepeatingJob` API added in the first alpha version of the library with this functionality, and now removed from the library, was implemented like this:

```kotlin
public fun LifecycleOwner.addRepeatingJob(
    state: Lifecycle.State,
    coroutineContext: CoroutineContext = EmptyCoroutineContext,
    block: suspend CoroutineScope.() -> Unit
): Job = lifecycleScope.launch(coroutineContext) {
    repeatOnLifecycle(state, block)
}
```

Given a `LifecycleOwner`, you could run a suspend block that restarts whenever its lifecycle moves in and out of the target state. This API uses the `LifecycleOwner`’s `lifecycleScope` to trigger a new coroutine and call `repeatOnLifecycle` inside it.

The code above would look like this using the `addRepeatingJob` API:

```kotlin
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleOwner.addRepeatingJob(Lifecycle.State.STARTED) {
            someLocationProvider.locations.collect {
                // New location! Update the map
            }
        }
    }
}
```

At first glance, you might think that this code is cleaner and requires less code. However, there are hidden gotchas that can make you shoot yourself in the foot if you don’t pay close attention:

* Even though `addRepeatingJob` takes a suspend block, `addRepeatingJob` is **NOT** a suspend function. Thus, you shouldn’t call it inside a coroutine!!!

* Less code? You only save one line of code with the cost of having a more error-prone API.

The first point might seem obvious but it always bites developers. And ironically, it’s actually based on one of the most conceptually core concepts of coroutines: [**Structured Concurrency**](https://elizarov.medium.com/structured-concurrency-722d765aa952).

`addRepeatingJob` is not a suspend function, and therefore, doesn’t support structured concurrency by default (note that you could manually make it support it by using the `coroutineContext` that it takes as a parameter). Since the `block` parameter is a suspend lambda, you relate this API to coroutines and you could easily write dangerous code like this:

```kotlin
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val job = lifecycleScope.launch {

            doSomeSuspendInitWork()

            // DANGEROUS! This API doesn't preserve the calling Context!
            // It won't get cancelled when the parent coroutine is cancelled!
            addRepeatingJob(Lifecycle.State.STARTED) {
                someLocationProvider.locations.collect {
                    // New location! Update the map
                }
            }
        }

        // If something goes wrong, cancel the coroutine launched above
        try {
            /* ... */
        } catch(t: Throwable) {
            job.cancel()
        }
    }
}
```

What’s wrong with this code? `addRepeatingJob` does coroutines stuff, nothing prevents me from calling it inside a coroutine, right?

As `addRepeatingJob` creates new coroutines to run the repeating block using `lifecycleScope` which is implicit in the implementation details, the new coroutines don’t respect structured concurrency nor preserve the calling coroutine context. Therefore, will NOT get canceled when you call `job.cancel()`. **This can lead to very subtle bugs in your app which are really difficult to debug**.

## repeatOnLifecycle FTW

The implicit `CoroutineScope` used inside `addRepeatingJob` is what makes the API unsafe to use in certain situations. It’s the hidden gotcha that requires extra attention to write correct code. This point is the recurring argument to avoid additional wrapper APIs on top of `repeatOnLifecycle` in the library.

The main benefit of the suspend `repeatOnLifecycle` API is that it cooperates with structured concurrency by default, whereas `addRepeatingJob` did not. It also helps you think in which scope you want the repeating job to happen. The API is self-explanatory and meets developer expectations:

* As any other suspend function, it will suspend the execution of the coroutine until something happens. In this case, until the lifecycle is destroyed.

* No surprises! It can be used in conjunction with other coroutines code and it’ll behave as you expect.

* The code surrounding `repeatOnLifecycle` is readable and makes sense for newcomers: *“First, I launch a new coroutine that follows the UI lifecycle. Then, I call repeatOnLifecycle that launches this block every time the UI reaches this lifecycle state”*.

## Flow.flowWithLifecycle

The `Flow.flowWithLifecycle` operator ([implementation here](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:lifecycle/lifecycle-runtime-ktx/src/main/java/androidx/lifecycle/FlowExt.kt;l=87)) is built on top of `repeatOnLifecycle` and only emits items sent by the upstream flow whenever the lifecycle is at least at `minActiveState`.

```kotlin
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        lifecycleScope.launch {
            someLocationProvider.locations
                .flowWithLifecycle(lifecycle, STARTED)
                .collect {
                    // New location! Update the map
                }
        }
    }
}
```

Even though this API also comes with some gotchas to be aware of, we decided to keep it in as it’s useful as a Flow operator. For example, it can be [easily used in Jetpack Compose](https://manuelvivo.dev/coroutines-addrepeatingjob#safe-flow-collection-in-jetpack-compose). Even though you could achieve the same functionality in Compose by using the [`produceState`](https://developer.android.com/jetpack/compose/side-effects#producestate) and the `repeatOnLifecycle` API, we left this API in the library as an alternative to a more reactive approach.

The gotcha, as it’s documented in the KDoc, is that the order in which you add the `flowWithLifecycle` operator matters. Operators applied before the `flowWithLifecycle` operator will be cancelled when the `lifecycle` is below `minActiveState`. However, operators applied *after* won’t be cancelled even though no items are sent.

For the most curious ones, this API name takes the `Flow.flowOn(CoroutineContext)` operator as a precedent since `Flow.flowWithLifecycle` changes the `CoroutineContext` used to collect the upstream flow while leaving the downstream unaffected.

## Should we add an additional API?

Given that we already have the `Lifecycle.repeatOnLifecycle`, `LifecycleOwner.repeatOnLifecycle`, and `Flow.flowWithLifecycle` APIs. Should we add any other API?

New APIs can introduce as much confusion as problems they’d solve. There are multiple ways to support different use cases, and the shortest path depends greatly on the surrounding code. What might work for your project, might not work for others.

This is why we don’t want to provide APIs for all the possible cases, the more APIs available, the more confusing it will be for developers to know *what* to use *when*. Therefore, we made the decision of just keeping the most low-level APIs. Sometimes, less is more.

## Naming is important (and difficult)

It’s not only about which use cases we support, but also, how to name them! Names should comply with developers’ expectations and follow the Kotlin coroutines conventions. For example:

* If the API starts a new coroutine using an implicit `CoroutineScope` (for example the `lifecycleScope` used implicitly in `addRepeatingJob`), this must be reflected in the name to avoid false expectations! In this case, `launch` should be somehow included in the name.

* `collect` is a suspend function. Don’t prefix an API name with `collect` if it’s not a suspend function.

> Note: Compose’s `collectAsState` API is a special case whose name we’re ok with. It cannot be confused for a suspend function since there’s no such thing as a `@Composable suspend fun` in Compose.

Even the `LifecycleOwner.addRepeatingJob` API was a tough one to name. As it creates new coroutines with `lifecycleScope`, it should’ve been prefixed with `launch`. However, we wanted to disassociate the fact that coroutines were used under the hood, and since it adds a new lifecycle observer, the name was more consistent with the rest of other LifecycleOwner APIs.

The name was also somewhat influenced by the existing [`LifecycleCoroutineScope.launchWhenX`](https://developer.android.com/reference/kotlin/androidx/lifecycle/LifecycleCoroutineScope) suspending APIs. As `launchWhenStarted` and `repeatOnLifecycle(STARTED)` provide completely different functionality (`launchWhenStarted` suspends the execution of the coroutine, and `repeatOnLifecycle` cancels and restarts a new coroutine), if the names of the new APIs were similar (for example, using `launchWhenever` for the restarting APIs), developers could’ve got confused and even use them interchangeably without noticing.

## One-liner flow collection

`LiveData`’s `observe` function is lifecycle aware and only processes emissions when the lifecycle is at least started. If you’re migrating from LiveData to Kotlin flows, you might think that having a one-line replacement is a good idea! You could remove boilerplate code and the migration is straightforward.

As such, you can do as [Ian Lake](https://twitter.com/ianhlake) did when he first started playing around with the `repeatOnLifecycle` APIs. He created a convenience wrapper called `collectIn` like the following (to follow the naming conventions discussed above, I’m renaming it to be `launchAndCollectIn`):

```kotlin
inline fun <T> Flow<T>.launchAndCollectIn(
    owner: LifecycleOwner,
    minActiveState: Lifecycle.State = Lifecycle.State.STARTED,
    crossinline action: suspend CoroutineScope.(T) -> Unit
) = owner.lifecycleScope.launch {
        owner.repeatOnLifecycle(minActiveState) {
            collect {
                action(it)
            }
        }
    }
```

So that you could call it from the UI like this:

```kotlin
class LocationActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        someLocationProvider.locations.launchAndCollectIn(this, STARTED) {
            // New location! Update the map
        }
    }
}
```

This wrapper, as nice and straightforward it might look in this example, suffers from the same problems we mentioned earlier regarding `LifecycleOwner.addRepeatingJob`. It doesn’t respect the calling context and can be dangerous to use inside other coroutines. Furthermore, the original name is really misleading: `collectIn` is not a suspend function! As mentioned before, developers expect `collect` functions to suspend. Maybe, a better name for this wrapper could be `Flow.launchAndCollectIn` to prevent bad usages.

## Wrapper in iosched

`repeatOnLifecycle` must be used with the `viewLifecycleOwner` in Fragments. In the open source Google I/O app, the [iosched](https://github.com/google/iosched) project, the team decided to create a wrapper to avoid misusages of the API in Fragments with a very explicit API name: [`Fragment.launchAndRepeatWithViewLifecycle`](https://github.com/google/iosched/blob/main/mobile/src/main/java/com/google/samples/apps/iosched/util/UiUtils.kt#L60).

> Note: The implementation is very similar to the `addRepeatingJob` API. And when this was written using the `alpha01` version of the library, the `repeatOnLifecycle` API lint checks that were added in `alpha02` were not in place.

## Do you need a wrapper?

If you need to create wrappers on top of the `repeatOnLifecycle` API to accommodate the most common use cases you might have in your app, ask yourself if you really need it, and why you need it. If you’re convinced and want to go forward, I’d suggest you choose a very explicit API name to clearly define what’s the wrapper’s behavior to avoid misusages. Also, document it very clearly so that newcomers can fully understand the implications of using it.

---

I hope this blog post gave you an idea of what considerations the team had when deciding what to do with the `repeatOnLifecycle` APIs and the potential helper methods we could add on top of it.

Again, thanks to [Adam Powell](https://twitter.com/adamwp), [Wojtek Kaliciński](https://twitter.com/wkalic), [Ian Lake](https://twitter.com/ianhlake), and [Yigit Boyar](https://twitter.com/yigitboyar) for giving feedback and discussing the shape of these APIs.
