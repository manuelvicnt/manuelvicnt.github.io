---
layout: post
current: post
cover: assets/images/2021-05-07-sharein-statein.png
navigation: True
title: Things to know about Flow’s shareIn and stateIn operators
date: 2021-05-07 00:00:00
tags: [coroutines]
class: post-template
subclass: 'post'
author: manuel
---

Become familiar with the `shareIn` and `stateIn` operators by example.

The [`Flow.shareIn`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/share-in.html) and [`Flow.stateIn`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/state-in.html) operators convert cold flows into hot flows: they can multicast the information that comes from a cold upstream flow to multiple collectors. They’re often used to improve performance, add a buffer when collectors are not present, or even as a caching mechanism.

> **Cold flows** are created on-demand and emit data when they’re being observed. **Hot flows** are always active and can emit data regardless of whether or not they’re being observed.

In this blog post, you’ll become familiar with the `shareIn` and `stateIn` operators by example. You’ll learn how to configure them to perform certain use cases and avoid common pitfalls you might encounter.

## The underlying flow producer

Continuing with the example from my [previous blog post](https://manuelvivo.dev/coroutines-addrepeatingjob), the underlying flow producer that we’re using emits location updates. It’s a *cold* flow, as it’s implemented using a [`callbackFlow`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/callback-flow.html). Every new collector will trigger the flow producer block, and a new callback will be added to the `FusedLocationProviderClient`.

```kotlin
class LocationDataSource(
    private val locationClient: FusedLocationProviderClient
) {
    val locationsSource: Flow<Location> = callbackFlow<Location> {
        val callback = object : LocationCallback() {
            override fun onLocationResult(result: LocationResult?) {
                result ?: return
                try { offer(result.lastLocation) } catch(e: Exception) {}
            }
        }
        requestLocationUpdates(createLocationRequest(), callback, Looper.getMainLooper())
            .addOnFailureListener { e ->
                close(e) // in case of exception, close the Flow
            }
        // clean up when Flow collection ends
        awaitClose {
            removeLocationUpdates(callback)
        }
    }
}
```

Let’s see how we can use the `shareIn` and `stateIn` operators to optimize the `locationsSource` flow for different use cases.

## shareIn or stateIn?

The first topic we’ll cover is the difference between `shareIn` and `stateIn`. The `shareIn` operator returns a [`SharedFlow`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-shared-flow/) instance whereas `stateIn` returns a [`StateFlow`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/index.html).

> Note: To learn more about `StateFlow` and `SharedFlow`, check out [our documentation](https://developer.android.com/kotlin/flow/stateflow-and-sharedflow).

`StateFlow` is a specialized configuration of `SharedFlow` optimized for sharing state: the last emitted item is replayed to new collectors, and items are conflated using [`Any.equals`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-any/equals.html). You can read more about this in the [`StateFlow` documentation](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-state-flow/index.html).

The main difference between these APIs is that the `StateFlow` interface allows you to access the last emitted value synchronously by reading its `value` property. That’s not the case with `SharedFlow`.

## Improving performance

These APIs can improve performance by sharing the same instance of the flow to be observed by all collectors instead of creating new instances of the same flow on-demand.

In the following example, `LocationRepository` consumes the `locationsSource` flow exposed by the `LocationDataSource` and applies the shareIn operator to make everyone interested in the user’s location collect from the same instance of the flow. Only one instance of the `locationsSource` flow is created and shared for all collectors:

```kotlin
class LocationRepository(
    private val locationDataSource: LocationDataSource,
    private val externalScope: CoroutineScope
) {
    val locations: Flow<Location> = 
        locationDataSource.locationsSource.shareIn(externalScope, WhileSubscribed())
}
```

The [`WhileSubscribed`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines.flow/-sharing-started/-while-subscribed.html) sharing policy is used to cancel the upstream flow when there are no collectors. In this way, we avoid wasting resources when no one is interested in location updates.

> **Tip for Android apps!** You can use **`WhileSubscribed(5000)`** most of the time to keep the upstream flow active for 5 seconds more after the disappearance of the last collector. That avoids restarting the upstream flow in certain situations such as configuration changes. This tip is especially helpful when upstream flows are expensive to create and when these operators are used in ViewModels.

## Buffering events

For this example, our requirements have changed, and now we’re asked to always listen for location updates and display the last 10 locations on the screen when the app comes from the background:

```kotlin
class LocationRepository(
    private val locationDataSource: LocationDataSource,
    private val externalScope: CoroutineScope
) {
    val locations: Flow<Location> = 
        locationDataSource.locationsSource
            .shareIn(externalScope, SharingStarted.Eagerly, replay = 10)
} 
```

We use a `replay` value of 10 to keep the last 10 emitted items in memory and re-emit those every time a collector observes the flow. To keep the underlying flow active all the time and emitting location updates, use the `SharingStarted.Eagerly` policy to listen for updates even if there are no collectors.

## Caching data

Our requirements have changed again, and in this case, we don’t need to be *always* listening for location updates if the app is in the background. However, we need to cache the last emitted item so that the user always sees some data on the screen, even if stale, while getting the current location. For this case, we can use the `stateIn` operator.

```kotlin
class LocationRepository(
    private val locationDataSource: LocationDataSource,
    private val externalScope: CoroutineScope
) {
    val locations: Flow<Location> = 
        locationDataSource.locationsSource.stateIn(externalScope, WhileSubscribed(), EmptyLocation)
}
```

`Flow.stateIn` caches and replays the last emitted item to a new collector.

## WATCH OUT! Do not create new instances on each function call

**NEVER** use `shareIn` or `stateIn` to create a new flow that’s returned when calling a function. That’d create a new `SharedFlow` or `StateFlow` on each function invocation that will remain in memory until the scope is cancelled or is garbage collected when there are no references to it.

```kotlin
class UserRepository(
    private val userLocalDataSource: UserLocalDataSource,
    private val externalScope: CoroutineScope
) {
    // DO NOT USE shareIn or stateIn in a function like this.
    // It creates a new SharedFlow/StateFlow per invocation which is not reused!
    fun getUser(): Flow<User> =
        userLocalDataSource.getUser()
            .shareIn(externalScope, WhileSubscribed())    

    // DO USE shareIn or stateIn in a property
    val user: Flow<User> = 
        userLocalDataSource.getUser().shareIn(externalScope, WhileSubscribed())
}
```

## Flows that require input

Flows that require input, like a `userId`, cannot be shared easily using `shareIn` or `stateIn`. Taking as an example the [iosched](https://github.com/google/iosched) open-source project — Google I/O’s Android app — the flow to get user events from [Firestore](https://firebase.google.com/docs/firestore/quickstart) is implemented using a `callbackFlow`, as you can see [in the source code](https://github.com/google/iosched/blob/main/shared/src/main/java/com/google/samples/apps/iosched/shared/data/userevent/FirestoreUserEventDataSource.kt#L107). As it takes the `userId` as a parameter, this flow cannot be reused easily using the `shareIn` or `stateIn` operators.

```kotlin
class UserRepository(
    private val userEventsDataSource: FirestoreUserEventDataSource
) {
    // New collectors will register as a new callback in Firestore.
    // As this function depends on a `userId`, the flow cannot be
    // reused by calling shareIn or stateIn in this function.
    // That will cause a new Shared/StateFlow to be created
    // every time the function is called.
    fun getUserEvents(userId: String): Flow<UserEventsResult> =
        userLocalDataSource.getObservableUserEvents(userId)
}
```

Optimizing this use case depends on the requirements of your app:

* Do you allow receiving events from multiple users at the same time? You might need to create a map of `SharedFlow`/`StateFlow` instances, and remove the reference and cancel the upstream flow when the `subscriptionCount` reaches zero.
* If you allow only one user, and all collectors need to update to the new user, you could emit event updates to a common `SharedFlow`/`StateFlow` for all collectors and use the common flow as a variable in the class.

---

The `shareIn` and `stateIn` operators can be used with cold flows to improve performance, add a buffer when collectors are not present, or even as a caching mechanism! Use them wisely, and don’t create new instances on each function call — it won’t work as you’d expect!
