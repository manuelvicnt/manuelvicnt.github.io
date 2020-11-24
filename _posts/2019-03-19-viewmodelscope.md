---
layout: post
current: post
cover: assets/images/2019-03-19-viewmodelscope.png
navigation: True
title: Easy Coroutines in Android - viewModelScope
date: 2019-03-19 00:00:00
tags: [coroutines]
class: post-template
subclass: 'post'
author: manuel
---

Learn everything you should know about viewModelScope

Cancelling coroutines when they are no longer needed can be a task easy to forget, it’s monotonous work and adds a lot of boilerplate code. `viewModelScope` contributes to [structured concurrency](https://kotlinlang.org/docs/reference/coroutines/basics.html#structured-concurrency) by adding an [extension property](https://kotlinlang.org/docs/reference/extensions.html#extension-properties) to the ViewModel class that automatically cancels its child coroutines when the ViewModel is destroyed.

## Scopes in ViewModels

A [`CoroutineScope`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-scope/) keeps track of all coroutines it creates. Therefore, if you cancel a scope, you cancel all coroutines it created. This is particularly important if you’re running coroutines in a [ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel). If your ViewModel is getting destroyed, all the asynchronous work that it might be doing must be stopped. Otherwise, you’ll waste resources and potentially leaking memory. If you consider that certain asynchronous work should persist after ViewModel destruction, it is because it should be done in a lower layer of your app’s architecture.

Add a `CoroutineScope` to your ViewModel by creating a new scope with a [`SupervisorJob`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-supervisor-job.html) that you cancel in the `onCleared()` method. The coroutines created with that scope will live as long as the ViewModel is being used. See following code:

```kotlin
class MyViewModel : ViewModel() {

    /**
     * This is the job for all coroutines started by this ViewModel.
     * Cancelling this job will cancel all coroutines started by this ViewModel.
     */
    private val viewModelJob = SupervisorJob()
    
    /**
     * This is the main scope for all coroutines launched by MainViewModel.
     * Since we pass viewModelJob, you can cancel all coroutines 
     * launched by uiScope by calling viewModelJob.cancel()
     */
    private val uiScope = CoroutineScope(Dispatchers.Main + viewModelJob)
    
    /**
     * Cancel all coroutines when the ViewModel is cleared
     */
    override fun onCleared() {
        super.onCleared()
        viewModelJob.cancel()
    }
    
    /**
     * Heavy operation that cannot be done in the Main Thread
     */
    fun launchDataLoad() {
        uiScope.launch {
            sortList() // happens on the background
            // Modify UI
        }
    }
    
    // Move the execution off the main thread using withContext(Dispatchers.Default)
    suspend fun sortList() = withContext(Dispatchers.Default) {
        // Heavy work
    }
}
```

The heavy work happening in the background will be cancelled if the ViewModel gets destroyed because the coroutine was started by that particular `uiScope`.

But that’s a lot of code to be included in every ViewModel, right? **`viewModelScope`** comes to simplify all this.

## viewModelScope means less boilerplate code

[AndroidX lifecycle v2.1.0](https://developer.android.com/jetpack/androidx/releases/lifecycle) introduced the extension property `viewModelScope` to the ViewModel class. It manages the coroutines in the same way we were doing in the previous section. That code is cut down to this:

```kotlin
class MyViewModel : ViewModel() {
  
    /**
     * Heavy operation that cannot be done in the Main Thread
     */
    fun launchDataLoad() {
        viewModelScope.launch {
            sortList()
            // Modify UI
        }
    }
  
    suspend fun sortList() = withContext(Dispatchers.Default) {
        // Heavy work
    }
}
```

All the `CoroutineScope` setup and cancellation is done for us. To use it, import the following dependency in your `build.gradle` file:

```groovy
implementation "androidx.lifecycle.lifecycle-viewmodel-ktx$lifecycle_version"
```

Let’s take a look at what’s happening under the hood.

## Digging into viewModelScope

The code is [publicly available](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:lifecycle/lifecycle-viewmodel-ktx/src/main/java/androidx/lifecycle/ViewModel.kt). `viewModelScope` is implemented as follows:

```kotlin
private const val JOB_KEY = "androidx.lifecycle.ViewModelCoroutineScope.JOB_KEY"

val ViewModel.viewModelScope: CoroutineScope
    get() {
        val scope: CoroutineScope? = this.getTag(JOB_KEY)
        if (scope != null) {
            return scope
        }
        return setTagIfAbsent(JOB_KEY,
            CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate))
    }
```

The ViewModel class has a `ConcurrentHashSet` attribute where it can store any kind of object. The CoroutineScope is stored there. If we take a look at the code, the method `getTag(JOB_KEY)` tries to retrieve the scope from there. If it doesn’t exist, then it creates a new CoroutineScope the same way we did before and adds the tag to the bag.

When the ViewModel is cleared, it executes the method `clear()` before calling the `onCleared()` method that we would’ve had to override otherwise. In the `clear()` method the ViewModel cancels the `Job` of the `viewModelScope`. The [full ViewModel code](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-master-dev:lifecycle/lifecycle-viewmodel/src/main/java/androidx/lifecycle/ViewModel.java) is also available but we are just focusing on the parts we are interested in:

```java
@MainThread
final void clear() {
    mCleared = true;
    // Since clear() is final, this method is still called on mock 
    // objects and in those cases, mBagOfTags is null. It'll always 
    // be empty though because setTagIfAbsent and getTag are not 
    // final so we can skip clearing it
    if (mBagOfTags != null) {
        for (Object value : mBagOfTags.values()) {
            // see comment for the similar call in setTagIfAbsent
            closeWithRuntimeException(value);
        }
    }
    onCleared();
}
```

The method goes through all the items in the bag and calls `closeWithRuntimeException` that checks if the object is of type `Closeable` and if so, closes it. In order for the ViewModel to close the scope, it needs to implement the `Closeable` interface. That’s why `viewModelScope` is of type `CloseableCoroutineScope` that extends `CoroutineScope` overriding the `coroutineContext` and implements the `Closeable` interface.

```kotlin
internal class CloseableCoroutineScope(
    context: CoroutineContext
) : Closeable, CoroutineScope {
  
    override val coroutineContext: CoroutineContext = context
  
    override fun close() {
        coroutineContext.cancel()
    }
}
```

## Dispatchers.Main as default

`Dispatchers.Main.immediate` is set as the default [`CoroutineDispatcher`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/index.html) for `viewModelScope`.

```kotlin
val scope = CloseableCoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
```

`Dispatchers.Main` is a natural fit for this case since ViewModel is a concept related to UI that is often involved in updating it so launching on another dispatcher will introduce at least 2 extra thread switches. Considering that suspend functions will do their own thread confinement properly, going with other Dispatchers wouldn’t be an option since we’d be making an assumption of what the ViewModel is doing.

`immediate` is used to execute the coroutine immediately without needing to re-[`dispatch`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/-coroutine-dispatcher/dispatch.html) the work to the appropriate thread.

## Unit Testing viewModelScope

`Dispatchers.Main` uses the `Android Looper.getMainLooper()` method to run code in the UI thread. That method is available in Instrumented Android tests but not in Unit tests.

Use the `org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutines_version` library to replace the Coroutines Main Dispatcher by calling `Dispatchers.setMain(dispatcher: CoroutineDispatcher)` with a `TestCoroutineDispatcher` that is available in `org.jetbrains.kotlinx:kotlinx-coroutines-test:$coroutines_version`. Note that `Dispatchers.setMain` is only needed if you use `viewModelScope` or you hardcode `Dispatchers.Main` in your codebase.

[`TestCoroutineDispatcher`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-test/kotlinx.coroutines.test/-test-coroutine-dispatcher/) is a dispatcher that gives us control of how coroutines are executed, being able to pause/resume execution and control its virtual clock. It was added as an experimental API in Kotlin Coroutines v1.2.1. You can read more about it in the [documentation](https://github.com/Kotlin/kotlinx.coroutines/tree/master/kotlinx-coroutines-test).

Don’t use `Dispatchers.Unconfined` as a replacement of Dispatchers.Main, it will break all assumptions and timings for code that does use `Dispatchers.Main`. Since a unit test should run well in isolation and without any side effects, you should call `Dispatchers.resetMain()` and clean up the executor when the test finishes running.

You can use this JUnitRule with that logic to simplify your code:

```kotlin
@ExperimentalCoroutinesApi
class CoroutinesTestRule(
        val testDispatcher: TestCoroutineDispatcher = TestCoroutineDispatcher()
) : TestWatcher() {

    override fun starting(description: Description?) {
        super.starting(description)
        Dispatchers.setMain(testDispatcher)
    }

    override fun finished(description: Description?) {
        super.finished(description)
        Dispatchers.resetMain()
        testDispatcher.cleanupTestCoroutines()
    }
}
```

Now, you can use it in your Unit Tests.

```kotlin
class MainViewModelUnitTest {
  
    @get:Rule
    var coroutinesTestRule = CoroutinesTestRule()
  
    @Test
    fun test() {
        /* ... */
    }
}
```

### Testing coroutines using Mockito

Do you use Mockito and want to `verify` that interactions with an object happen? Note that using Mockito’s `verify` method is not the preferred way to unit test your code. You should check app-specific logic such as an element is present rather than verifying that interactions with an object happen.

Before checking that the interaction with an object happened, we need to make sure that all coroutines launched have finished. Let’s take a look at the following example.

```kotlin
class MainViewModel(private val dependency: Any): ViewModel {
  
  fun sampleMethod() {
    viewModelScope.launch {
      val hashCode = dependency.hashCode()
      // TODO: do something with hashCode
  }
}

class MainViewModelUnitTest {

  // Mockito setup goes here
  /* ... */
  
  @get:Rule
  var coroutinesTestRule = CoroutinesTestRule()
  
  @Test
  fun test() = coroutinesTestRule.testDispatcher.runBlockingTest {
    val subject = MainViewModel(mockObject)
    subject.sampleMethod()
    // Checks mockObject called the hashCode method that is expected from the coroutine created in sampleMethod
    verify(mockObject).hashCode()
  }
}
```

In the test, we call the `runBlockingTest` method inside the `TestCoroutineDispatcher` that the rule creates. Since that Dispatcher overrides `Dispatchers.Main`, MainViewModel will run the coroutine on that Dispatcher too. Calling `runBlockingTest` will make that coroutine to execute synchronously in the test. Since our `verify` Mockito call is inside the `runBlockingTest` block, it will be called after the coroutine finishes and the interaction will have happened at that moment.

For another example, check out how we added this kind of Unit tests to the Kotlin Coroutines codelab in the [this PR](https://github.com/googlecodelabs/kotlin-coroutines/pull/29).

---

If you are using architecture components, ViewModel and coroutines, use `viewModelScope` to let the framework manage its lifecycle for you. It’s a no brainer!

The [Coroutines codelab](https://codelabs.developers.google.com/codelabs/kotlin-coroutines) has been already updated to use it. Check it out to find out more about Coroutines and how to use them in an Android app.
