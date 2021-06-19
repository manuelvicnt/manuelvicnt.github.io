---
layout: post
current: post
cover: assets/images/2021-06-10-coroutinescope-hilt.png
navigation: True
title: Create an application CoroutineScope using Hilt
date: 2021-06-10 00:00:00
tags: [coroutines, hilt]
class: post-template
subclass: 'post'
author: manuel
---

Inject an application-scoped `CoroutineScope` using Hilt.

Following [coroutine’s best practices](https://developer.android.com/kotlin/coroutines/coroutines-best-practices), you might need to inject an application-scoped `CoroutineScope` in some classes to [launch new coroutines that follow the app lifecycle or to make certain work outlive the caller’s scope](https://developer.android.com/kotlin/coroutines/coroutines-best-practices#create-coroutines-data-layer).

In this article, you’ll learn how to create an application-scoped `CoroutineScope` using Hilt, and how to inject it as a dependency. To further improve the way we work with Coroutines, we’ll see how to inject the different `CoroutineDispatcher`s and replace their implementations in tests.

## Manual dependency injection

To create an [application-scoped](https://medium.com/androiddevelopers/scoping-in-android-and-hilt-c2e5222317c0) `CoroutineScope` following dependency injection (DI) best practices [manually](https://developer.android.com/training/dependency-injection/manual) without any library, you’d typically add a new variable to your application class with an instance of a `CoroutineScope`. The same instance would be manually passed around when creating other objects.

```kotlin
class MyRepository(private val externalScope: CoroutineScope) { /* ... */ }

class MyApplication : Application() {

    // Application-scoped types that any class in the app could access
    // using the applicationContext.
    val applicationScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
    val myRepository = MyRepository(applicationScope)

}
```

Since there isn’t a reliable way to know when the `Application` is destroyed in Android, you don’t need to call `applicationScope.cancel()` manually as the scope and all ongoing work will be destroyed when the application process finishes.

A better option for doing this manually is to create an `ApplicationContainer` class that holds the application-scoped types. This helps with separation of concerns since these _Container_ classes are responsible for:

* handling the logic of _how_ to build certain types,
* holding container-scoped types instances, and
* returning instances of scoped and unscoped types.

```kotlin
class ApplicationDiContainer {
    val applicationScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)
    val myRepository = MyRepository(applicationScope)
}

class MyApplication : Application() {
    val applicationDiContainer = ApplicationDiContainer()
}
```

> Note: A container always returns the same instance of a scoped type, and always returns a different instance for unscoped types. Scoping types to containers is [costly](https://developer.android.com/training/dependency-injection/hilt-android#component-scopes) since the scoped object stays in memory until the component is destroyed, so only scope what’s really needed.

In the `ApplicationDiContainer` example above, all types were scoped. If `MyRepository` didn’t need to be scoped to the application, we’d have:

```kotlin
class ApplicationDiContainer {
    // Scoped type. Same instance is always returned
    val applicationScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

    // Unscoped type. Always returns a different instance
    fun getMyRepository(): MyRepository {
        return MyRepository(applicationScope)
    }
}
```

## Using Hilt in your app

Hilt generates what you can see in `ApplicationDiContainer` (and more!) at compile time using annotations. Moreover, Hilt provides containers for [most Android framework classes](https://developer.android.com/training/dependency-injection/hilt-android#generated-components) not only for the `Application` class.

To set up Hilt in your app and create the container for the `Application` class, annotate your `Application` class with `@HiltAndroidApp`.

```kotlin
@HiltAndroidApp
class MyApplication : Application()
```

With this, the application DI container is ready to be used. We just need to let Hilt know how to provide instances of different types.

> Note: In Hilt, Container classes are referenced as Components. The container associated with the `Application` class is called `SingletonComponent`. Check out the [list of all available Hilt components](https://developer.android.com/training/dependency-injection/hilt-android#generated-components).

## Construction injection

Construction injection is the easiest way to let Hilt know how to provide instances of a type if we have access to the constructor of a class as we only need to annotate the constructor with `@Inject`:

```kotlin
@Singleton // Scopes this type to the SingletonComponent
class MyRepository @Inject constructor(
   private val externalScope: CoroutineScope
) { 
    /* ... */ 
}
```

This lets Hilt know that in order to provide an instance of the `MyRepository` class, an instance of `CoroutineScope` needs to be passed as a dependency. Hilt generates code at compile time to make sure dependencies are satisfied and passed in when creating an instance of a type or give errors in case it doesn’t have enough information. `@Singleton` is used to scope this class to the `SingletonContainer`.

At this point, Hilt doesn’t know how to satisfy the `CoroutineScope` dependency because we haven’t told Hilt how to do that. The following sections will explain how we can let Hilt know what to pass as a dependency.

> Note: Hilt provides a different annotation to scope types to the different Hilt available components. Check out the [list of all available component scopes](https://developer.android.com/training/dependency-injection/hilt-android#component-scopes).

## Bindings

A _binding_ is a commonly-used term in Hilt that denotes the **information** Hilt knows about how to provide instances of a type as a dependency. We could say that we added a binding to Hilt with the `@Inject` annotation of the code snippet above.

Bindings flow through [Hilt’s components hierarchy](https://developer.android.com/training/dependency-injection/hilt-android#component-hierarchy). Bindings that are available in the `SingletonComponent` are also available in the `ActivityComponent`.

Bindings for unscoped types (an example of this could’ve been the `MyRepository` code above if it wasn’t annotated with `@Singleton`), are available in all Hilt components. Bindings that are scoped to a component, such as `MyRepository` that is annotated with `@Singleton`, are available to the scoped component and the components below it in the hierarchy.

## Providing types with modules

As mentioned above, we need to let Hilt know how to satisfy the `CoroutineScope` dependency. However, `CoroutineScope` is an interface type that comes from an external library, so we cannot use constructor injection as we did before with the `MyRepository` class. The alternative is letting Hilt know what code to run when providing the instance of a type [using Modules](https://developer.android.com/training/dependency-injection/hilt-android#hilt-modules):

```kotlin
@InstallIn(SingletonComponent::class)
@Module
object CoroutinesScopesModule {

    @Singleton // Provide always the same instance 
    @Provides
    fun providesCoroutineScope(): CoroutineScope {
        // Run this code when providing an instance of CoroutineScope
        return CoroutineScope(SupervisorJob() + Dispatchers.Default)
    }
}
```

[The `@Provides` method](https://developer.android.com/training/dependency-injection/hilt-android#inject-provides) is annotated with `@Singleton` to make Hilt always return the same instance of that `CoroutineScope`. This is because any work that needs to follow the application lifetime should be created using the same instance of a `CoroutineScope` that follows the `Application`’s lifecycle.

Hilt modules are annotated with `@InstallIn` that indicates in which Hilt component (and components below in the hierarchy) the binding is installed. In our case, as the application `CoroutineScope` is needed by `MyRepository` which is scoped to the `SingletonComponent`, this binding needs to be installed in the `SingletonComponent` as well.

In Hilt jargon, we could say that we added a `CoroutineScope` binding, as now, Hilt knows how to provide instances of `CoroutineScope`.

However, the code snippet above could be improved. [Hardcoding dispatchers is a bad practice in coroutines](https://developer.android.com/kotlin/coroutines/coroutines-best-practices#inject-dispatchers), we should inject them to **make them configurable and make testing easier**. Following the previous code, we can create a new Hilt module to let it know which Dispatcher to inject for each case: main, default, and IO.

## Providing implementations for CoroutineDispatcher

We have to provide different implementations for the same type: `CoroutineDispatcher`. In other words, we need different bindings for the same type.

We use [_qualifiers_](https://developer.android.com/training/dependency-injection/hilt-android#multiple-bindings) to let Hilt know which binding, or implementation, to use each time. Qualifiers are just annotations that you and Hilt use to identify specific bindings. Let’s create one qualifier per `CoroutineDispatcher` implementation:

```kotlin
// CoroutinesQualifiers.kt file

@Retention(AnnotationRetention.RUNTIME)
@Qualifier
annotation class DefaultDispatcher

@Retention(AnnotationRetention.RUNTIME)
@Qualifier
annotation class IoDispatcher

@Retention(AnnotationRetention.RUNTIME)
@Qualifier
annotation class MainDispatcher

@Retention(AnnotationRetention.BINARY)
@Qualifier
annotation class MainImmediateDispatcher
```

Then, these qualifiers annotate the different `@Provides` methods to identify a specific binding in Hilt modules. The `@DefaultDispatcher` qualifier annotates the method that returns the default dispatcher, and so on.

```kotlin
@InstallIn(SingletonComponent::class)
@Module
object CoroutinesDispatchersModule {

    @DefaultDispatcher
    @Provides
    fun providesDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default

    @IoDispatcher
    @Provides
    fun providesIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @MainDispatcher
    @Provides
    fun providesMainDispatcher(): CoroutineDispatcher = Dispatchers.Main

    @MainImmediateDispatcher
    @Provides
    fun providesMainImmediateDispatcher(): CoroutineDispatcher = Dispatchers.Main.immediate
}
```

Note that these `CoroutineDispatcher`s don’t need to be scoped to the `SingletonComponent`. Every time these dependencies are needed, Hilt calls the `@Provides` method and returns the corresponding `CoroutineDispatcher`.

## Providing an application-scoped CoroutineScope

To get rid of the hardcoded `CoroutineDispatcher` from our previous application-scoped `CoroutineScope` code, we need to inject the Hilt-provided default dispatcher. For that, we can pass in the type we want to inject, `CoroutineDispatcher`, using the corresponding qualifier, `@DefaultDispatcher`, as a dependency in the method that provides the application `CoroutineScope`.

```kotlin
@InstallIn(SingletonComponent::class)
@Module
object CoroutinesScopesModule {

    @Singleton
    @Provides
    fun providesCoroutineScope(
        @DefaultDispatcher defaultDispatcher: CoroutineDispatcher
    ): CoroutineScope = CoroutineScope(SupervisorJob() + defaultDispatcher)
}
```

As Hilt has multiple bindings for the `CoroutineDispatcher` type, we disambiguate it using the `@DefaultDispatcher` annotation when `CoroutineDispatcher` is used as a dependency.

## A qualifier for ApplicationScope

Even though we don’t need multiple bindings for `CoroutineScope` at the moment (this could change in the future if we ever need something like a `UserCoroutineScope`), adding a qualifier to the application `CoroutineScope` helps with readability when injecting it as a dependency.

```kotlin
@Retention(AnnotationRetention.RUNTIME)
@Qualifier
annotation class ApplicationScope

@InstallIn(SingletonComponent::class)
@Module
object CoroutinesScopesModule {

    @Singleton
    @ApplicationScope
    @Provides
    fun providesCoroutineScope(
        @DefaultDispatcher defaultDispatcher: CoroutineDispatcher
    ): CoroutineScope = CoroutineScope(SupervisorJob() + defaultDispatcher)
}
```

As `MyRepository` depends on this scope, it’s very clear which external scope uses as implementation:

```kotlin
@Singleton
class MyRepository @Inject constructor(
    @ApplicationScope private val externalScope: CoroutineScope
) { /* ... */ }
```

## Replacing Dispatchers for instrumentation tests

We said before that we should inject dispatchers to make testing easier and have full control over what’s happening. For instrumentation tests, we’d want to make Espresso wait for coroutines to finish.

Instead of creating a custom `CoroutineDispatcher` with some [Espresso Idling resource](https://developer.android.com/training/testing/espresso/idling-resource) to make it wait for the coroutines to finish, we can take advantage of the [AsyncTask](https://developer.android.com/reference/android/os/AsyncTask) API. Even though AsyncTask was deprecated in Android API 30, Espresso hooks into its thread pool to check for idleness. Therefore, any coroutine that should be executed in a background thread could be executed in the AsyncTask’s thread pool.

Use Hilt’s `TestInstallIn` API to make Hilt provide a different implementation of a type in tests. Similar to how we provided the different Dispatchers above, we can create a new file under the `androidTest` package to provide different implementations for those Dispatchers.

```kotlin
// androidTest/projectPath/TestCoroutinesDispatchersModule.kt file

@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [CoroutinesDispatchersModule::class]
)
@Module
object TestCoroutinesDispatchersModule {

    @DefaultDispatcher
    @Provides
    fun providesDefaultDispatcher(): CoroutineDispatcher =
        AsyncTask.THREAD_POOL_EXECUTOR.asCoroutineDispatcher()

    @IoDispatcher
    @Provides
    fun providesIoDispatcher(): CoroutineDispatcher =
        AsyncTask.THREAD_POOL_EXECUTOR.asCoroutineDispatcher()

    @MainDispatcher
    @Provides
    fun providesMainDispatcher(): CoroutineDispatcher = Dispatchers.Main
}
```

With the code above, we’re making Hilt “forget” the `CoroutinesDispatchersModule` used in production code in tests. That module will be replaced with `TestCoroutinesDispatchersModule` which uses the Async Task’s thread pool for work that needs to happen in the background, and `Dispatchers.Main` for work that needs to happen on the main thread which Espresso also waits for.

> Warning: This implementation is obviously a hack that we’re not proud of. However, coroutines don’t currently integrate well with Espresso as there isn’t a way to know if a `CoroutineDispatcher` is idle or not at the moment ([Link to bug](https://github.com/Kotlin/kotlinx.coroutines/issues/242)). `AsyncTask.THREAD_POOL_EXECUTOR` is the best alternative to use at the moment since Espresso doesn’t use Idling resources to check if this executor is idle, Espresso uses a different heuristic that takes into account what’s in the message queue. That makes it a better option than something like [`IdlingThreadPoolExecutor`](https://developer.android.com/reference/androidx/test/espresso/idling/concurrent/IdlingThreadPoolExecutor) which unfortunately considers the thread pool idle when a coroutine is suspended due to how coroutines are compiled down to a state machine.

For more information about testing, check out [Hilt’s testing guide](https://developer.android.com/training/dependency-injection/hilt-testing).

---

In this article, you learnt how to create an application-scoped `CoroutineScope` using Hilt, inject it as a dependency, inject the different `CoroutineDispatcher` instances, and replace their implementations in tests.
