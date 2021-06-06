---
layout: post
current: post
cover: assets/images/2019-03-24-suspend-modifier.png
navigation: True
title: The suspend modifier — under the hood
date: 2019-03-24 00:00:00
tags: [coroutines]
class: post-template
subclass: 'post'
author: manuel
---

How is the compiler transforming the code to be able to suspend and resume the execution of coroutines?

Kotlin coroutines introduced the ***suspend modifier*** in our daily life as Android developers. Are you curious to see what’s happening under the hood? How is the compiler transforming the code to be able to suspend and resume the execution of coroutines?

Knowing this will help you better understand why a suspend function won’t return until all the work that it started has completed and how the code can suspend without blocking threads.

> **TL;DR;** The Kotlin compiler will create a state machine for every suspend function that manages the coroutine’s execution for us!

📚 New to coroutines on Android? Check out these Coroutines codelabs:

* [Using coroutines in your Android app](https://codelabs.developers.google.com/codelabs/kotlin-coroutines/#0)

* [Advanced Coroutines with Kotlin Flow and Live Data](https://codelabs.developers.google.com/codelabs/advanced-kotlin-coroutines/#0)

If you prefer to watch a video about this, check this out:

<iframe width="560" height="315" src="https://www.youtube.com/embed/IQf-vtIC-Uc" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## Coroutines 101

Coroutines simplify asynchronous operations on Android. As explained in the [documentation](https://developer.android.com/kotlin/coroutines), we can use them to manage asynchronous tasks that might otherwise block the main thread and cause your app to freeze.

Coroutines are also helpful to replace callback-based APIs with imperative looking code. As example, check out this asynchronous code that uses callbacks:

```kotlin
// Simplified code that only considers the happy path
fun loginUser(userId: String, password: String, userResult: Callback<User>) {
  // Async callbacks
  userRemoteDataSource.logUserIn { user ->
    // Successful network request
    userLocalDataSource.logUserIn(user) { userDb ->
      // Result saved in DB
      userResult.success(userDb)
    }
  }
}
```

Those callbacks can be converted to sequential function calls using coroutines:

```kotlin
suspend fun loginUser(userId: String, password: String): User {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  return userDb
}
```

In the coroutines code, we added the **suspend** modifier to the function. That tells the compiler that this function needs to be executed inside a coroutine. As a developer, you can think of a suspend function as a regular function whose execution might be suspended and resumed at some point.

Unlike callbacks, coroutines provide an easy way to swap between threads and handle exceptions.

But, what’s the compiler actually doing under the hood when we mark the function as *suspend*?

## Suspend under the hood

Back to the `loginUser` suspend function, notice that the other functions it calls are also suspend functions:

```kotlin
suspend fun loginUser(userId: String, password: String): User {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  return userDb
}

// UserRemoteDataSource.kt
suspend fun logUserIn(userId: String, password: String): User

// UserLocalDataSource.kt
suspend fun logUserIn(userId: String): UserDb
```

In a nutshell, the Kotlin compiler will take suspend functions and convert them to an optimised version of callbacks using a [**finite state machine**](https://en.wikipedia.org/wiki/Finite-state_machine) (which we’ll cover later).

You got it right, **the compiler will write those callbacks for you**!

### Continuation interface

The way suspend functions communicate with each other is with `Continuation` objects. A [`Continuation`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/-continuation/index.html) is just a generic callback interface with some extra information. As we will see later, it will represent the generated state machine of a suspend function.

Let’s take a look at its definition:

```kotlin
interface Continuation<in T> {
  public val context: CoroutineContext
  public fun resumeWith(value: Result<T>)
}
```

* `context` will be the `CoroutineContext` to be used in that continuation.

* `resumeWith` resumes execution of the coroutine with a [`Result`](https://github.com/Kotlin/kotlinx.coroutines/blob/master/stdlib-stubs/src/Result.kt), that can contain either a value which is the result of the computation that caused the suspension or an exception.

Note: From Kotlin 1.3 onwards, you can also use the extension functions [`resume(value: T)`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/resume.html) and [`resumeWithException(exception: Throwable)`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/resume-with-exception.html) which are specialised versions of the `resumeWith` call.

The compiler will replace the suspend modifier with the extra parameter `completion` (of type `Continuation`) in the function signature that will be used to communicate the result of the suspend function to the coroutine that called it:

```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  completion.resume(userDb)
}
```

For simplicity, our example will return `Unit` instead of `User`. The `User` object will be “returned” in the added `Continuation` parameter.

The bytecode of suspend functions actually return `Any?` because it's a union type of `T | COROUTINE_SUSPENDED`. That allows the function to return synchronously when it can.

> Note: If you mark a function that doesn’t call other suspend functions with the suspend modifier, the compiler will add the extra Continuation parameter but won’t do anything with it, the function body’s bytecode will look like a regular function.

You can also see the `Continuation` interface in other places:

* When converting callback-based APIs to coroutines using [`suspendCoroutine`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/suspend-coroutine.html) or [`suspendCancellableCoroutine`](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/suspend-cancellable-coroutine.html) (which you should always prefer), you directly interact with a `Continuation` object to resume the coroutine that got suspended after running the block of code passed as a parameter.

* You can start a coroutine with the [`startCoroutine`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/start-coroutine.html) extension function on a suspend function. It takes a `Continuation` object as a parameter that will get called when the new coroutine finishes with either a result or an exception.

### Using different Dispatchers

You can swap between different Dispatchers to execute computations on different threads. How does Kotlin know where to resume a suspended computation?

There’s a subtype of `Continuation` called [`DispatchedContinuation`](https://github.com/Kotlin/kotlinx.coroutines/blob/master/kotlinx-coroutines-core/common/src/internal/DispatchedContinuation.kt) whose resume function makes a dispatch call to the `Dispatcher` available in the `CoroutineContext`. All `Dispatchers` will call dispatch except `Dispatchers.Unconfined` whose `isDispatchNeeded` function override (that is called before `dispatch`) always returns `false`.

## The generated State machine

> **Disclaimer**: The code to be shown in the rest of the article will not fully match the bytecode generated by the compiler. It will be Kotlin code accurate enough to allow you to understand what’s really happening internally. This representation is generated by Coroutines version 1.3.3 and might change in future versions of the library.

The Kotlin compiler will identify when the function can suspend internally. Every suspension point will be represented as a state in the finite state machine. These states are represented with labels by the compiler:

```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {
  
  // Label 0 -> first execution
  val user = userRemoteDataSource.logUserIn(userId, password)
  
  // Label 1 -> resumes from userRemoteDataSource
  val userDb = userLocalDataSource.logUserIn(user)
  
  // Label 2 -> resumes from userLocalDataSource
  completion.resume(userDb)
}
```

For a better representation of the state machine, the compiler will use a `when` statement to implement the different states:

```kotlin
fun loginUser(userId: String, password: String, completion: Continuation<Any?>) {
  when(label) {
    0 -> { // Label 0 -> first execution
        userRemoteDataSource.logUserIn(userId, password)
    }
    1 -> { // Label 1 -> resumes from userRemoteDataSource
        userLocalDataSource.logUserIn(user)
    }
    2 -> { // Label 2 -> resumes from userLocalDataSource
        completion.resume(userDb)
    }
    else -> throw IllegalStateException(/* ... */)
  }
}
```

This code is incomplete since the different states have no way to share information. The compiler will use the same `Continuation` object in the function to do it. This is why the generic of the `Continuation` is `Any?` instead of the return type of the original function (i.e. `User`).

Furthermore, the compiler will create a private class that 1) holds the required data and 2) calls the `loginUser` function recursively to resume execution. You can check out an approximation of the generated class below.

> **Disclaimer**: Comments are not generated by the compiler. I added them to explain what they do and make following the code easier to follow.

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
  
  class LoginUserStateMachine(
    // completion parameter is the callback to the function 
    // that called loginUser
    completion: Continuation<Any?>
  ): CoroutineImpl(completion) {
  
    // Local variables of the suspend function
    var user: User? = null
    var userDb: UserDb? = null
  
    // Common objects for all CoroutineImpls
    var result: Any? = null
    var label: Int = 0
  
    // this function calls the loginUser again to trigger the
    // state machine (label will be already in the next state) and
    // result will be the result of the previous state's computation
    override fun invokeSuspend(result: Any?) {
      this.result = result
      loginUser(null, null, this)
    }
  }
  /* ... */
}
```

As `invokeSuspend` will call `loginUser` again with just the information of the `Continuation` object, the rest of parameters in the `loginUser` function signature become nullable. At this point, the compiler just needs to add information on how to move between states.

The first thing it needs to do is know if 1) it’s the first time the function is called or 2) the function has resumed from a previous state. It does it by checking if the continuation passed in is of type `LoginUserStateMachine` or not:

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
  /* ... */
  val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)
  /* ... */
}
```

If it’s the first time, it will create a new `LoginUserStateMachine` instance and will store the `completion` instance received as a parameter so that it remembers how to resume the function that called this one. If it’s not, it will just carry on executing the state machine (the suspend function).

Now, let’s see the code that the compiler generates for moving between states and sharing information between them.

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
    /* ... */

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        0 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Next time this continuation is called, it should go to state 1
            continuation.label = 1
            // The continuation object is passed to logUserIn to resume 
            // this state machine's execution when it finishes
            userRemoteDataSource.logUserIn(userId!!, password!!, continuation)
        }
        1 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.user = continuation.result as User
            // Next time this continuation is called, it should go to state 2
            continuation.label = 2
            // The continuation object is passed to logUserIn to resume 
            // this state machine's execution when it finishes
            userLocalDataSource.logUserIn(continuation.user, continuation)
        }
          /* ... leaving out the last state on purpose */
    }
}
```

Spend some time going through the code above and see if you can spot the differences with the previous snippets of code. Let’s see what the compiler generates:

* The `when` statement’s argument is the `label` from inside the `LoginUserStateMachine` instance.

* Every time a new state is processed, there’s a check in case a failure happened when this function was suspended.

* Before calling the next suspend function (i.e. `logUserIn`), the `label` of the `LoginUserStateMachine` instance is updated to the next state.

* When inside this state machine there’s a call to another suspend function, the instance of the `continuation` (of type `LoginUserStateMachine`) is passed as a parameter. The suspend function to be called has also been transformed by the compiler and it’s another state machine like this one that takes a continuation object as a parameter! When the state machine of that suspend function finishes, it will resume the execution of this state machine.

The last state is different since it has to resume the execution of the function that called this one, as you can see in the code, it calls resume on the cont variable stored (at construction time) in `LoginUserStateMachine`:

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {
    /* ... */

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        /* ... */
        2 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.userDb = continuation.result as UserDb
            // Resumes the execution of the function that called this one
            continuation.cont.resume(continuation.userDb)
        }
        else -> throw IllegalStateException(/* ... */)
    }
}
```

As you see, the Kotlin compiler is doing a lot for us! From this suspend function:

```kotlin
suspend fun loginUser(userId: String, password: String): User {
  val user = userRemoteDataSource.logUserIn(userId, password)
  val userDb = userLocalDataSource.logUserIn(user)
  return userDb
}
```

The compiler generated all of this for us:

```kotlin
fun loginUser(userId: String?, password: String?, completion: Continuation<Any?>) {

    class LoginUserStateMachine(
        // completion parameter is the callback to the function that called loginUser
        completion: Continuation<Any?>
    ): CoroutineImpl(completion) {
        // objects to store across the suspend function
        var user: User? = null
        var userDb: UserDb? = null

        // Common objects for all CoroutineImpl
        var result: Any? = null
        var label: Int = 0

        // this function calls the loginUser again to trigger the 
        // state machine (label will be already in the next state) and 
        // result will be the result of the previous state's computation
        override fun invokeSuspend(result: Any?) {
            this.result = result
            loginUser(null, null, this)
        }
    }

    val continuation = completion as? LoginUserStateMachine ?: LoginUserStateMachine(completion)

    when(continuation.label) {
        0 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Next time this continuation is called, it should go to state 1
            continuation.label = 1
            // The continuation object is passed to logUserIn to resume 
            // this state machine's execution when it finishes
            userRemoteDataSource.logUserIn(userId!!, password!!, continuation)
        }
        1 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.user = continuation.result as User
            // Next time this continuation is called, it should go to state 2
            continuation.label = 2
            // The continuation object is passed to logUserIn to resume 
            // this state machine's execution when it finishes
            userLocalDataSource.logUserIn(continuation.user, continuation)
        }
        2 -> {
            // Checks for failures
            throwOnFailure(continuation.result)
            // Gets the result of the previous state
            continuation.userDb = continuation.result as UserDb
            // Resumes the execution of the function that called this one
            continuation.cont.resume(continuation.userDb)
        }
        else -> throw IllegalStateException(/* ... */)
    }
}
```

---

The Kotlin compiler transforms every suspend function to be a state machine, which optimises using callbacks every time a function needs to suspend.

Knowing what the compiler does under the hood now, you can better understand why a suspend function won’t return until all the work that it started has completed. Also, how the code can suspend without blocking threads: the information of what needs to be executed when the function is resumed is stored in the `Continuation` object!