---
layout: post
current: post
cover: assets/images/2020-hilt.png
navigation: True
title: Dependency injection on Android with Hilt
date: 2020-06-10 00:00:00
tags: [hilt]
class: post-template
subclass: 'post'
author: manuel
---

Learn about Jetpack's recommended library for dependency injection

[Dependency injection](https://developer.android.com/training/dependency-injection) (DI) is a technique widely used in programming and well suited to Android development, where dependencies are provided to a class instead of creating them itself. By following DI principles, you lay the groundwork for good app architecture, greater code reusability, and ease of testing. Have you ever tried manual dependency injection in your app? Even with many of the existing dependency injection libraries today, it requires a lot of boilerplate code as your project becomes larger, since you have to construct every class and its dependencies by hand, and create containers to reuse and manage dependencies.

> By following DI principles, you lay the groundwork for good app architecture, greater code reusability, and ease of testing.

The new [Hilt](https://developer.android.com/training/dependency-injection/hilt-android) library defines a **standard way** to do DI in your application by providing containers for every Android class in your project and managing their lifecycles *automatically* for you. Hilt is currently in *alpha*, try it in your app and give us feedback using this [link](https://github.com/google/dagger/issues/new).

Hilt is built on top of the popular DI library [Dagger](https://developer.android.com/training/dependency-injection/dagger-basics) so benefits from the **compile time correctness, runtime performance, scalability, and [Android Studio support](https://manuelvivo.dev/dagger-hilt-navigation-android-studio)** that Dagger provides. Due to this, Dagger’s seen great adoption on 30% of top 10k apps of the Google Play Store. However, because of the compile time code generation, expect a build time increase.

Since many Android framework classes are instantiated by the OS itself, there’s an associated boilerplate when using Dagger in Android apps. Unlike Dagger, Hilt is integrated with Jetpack libraries and Android framework classes and removes most of that boilerplate to let you **focus on just the important parts** of defining and injecting bindings without worrying about managing all of the Dagger setup and wiring. It automatically generates and provides:

* **Components for integrating Android framework classes** with Dagger that you would otherwise need to create by hand.

* **Scope annotations** for the components that Hilt generates automatically.

* **Predefined bindings and qualifiers**.

Best of all, **as Dagger and Hilt can coexist together, apps can be migrated on an as-needed basis**.

## Hilt in action

Just to show you how easy to use Hilt is, let’s perform some quick DI in a typical Android app. Let’s make Hilt inject an `AnalyticsAdapter` into our `MainActivity`.

First, enable Hilt in your app by annotating your application class with the `@HiltAndroidApp` to trigger Hilt’s code generation:

```kotlin
@HiltAndroidApp
class MyApplication : Application() {/* ... */}
```

Second, tell Hilt how to provide instances of `AnalyticsAdapter` by annotating its constructor with `@Inject`:

```kotlin
class AnalyticsAdapter @Inject constructor() {/* ... */}
```

And third, to inject an instance of `AnalyticsAdapter` into `MainActivity`, enable Hilt in the activity with the `@AndroidEntryPoint` annotation and perform field injection using the `@Inject` annotation:

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {
  
  @Inject lateinit var analytics: AnalyticsAdapter
  
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // analytics instance has been populated by Hilt
    // and it's ready to be used
  }
}
```

For more information, you can easily check out what the new annotations do in the cheat sheet section below.

## Comes with Jetpack support!

You can use your favourite Jetpack libraries with Hilt out of the box. We’re providing direct injection **support for ViewModel and WorkManager** in this release.

For example, to inject a [Architecture Components ViewModel](https://developer.android.com/topic/libraries/architecture/viewmodel) `LoginViewModel` into a `LoginActivity`: annotate `LoginViewModel` with `@ViewModelInject` and use it in the activity or fragment as you’d expect:

```kotlin
class LoginViewModel @ViewModelInject constructor(
  private val analyticsAdapter: AnalyticsAdapter
): ViewModel() { ... }

@AndroidEntryPoint
class LoginActivity : AppCompatActivity() {
  
  private val loginViewModel: LoginViewModel by viewModels()
  
  override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // loginViewModel is ready to be used
  }
}
```

Learn more about Jetpack support in the [docs](https://developer.android.com/training/dependency-injection/hilt-jetpack).

## Start using Hilt

If you’re intrigued by Hilt and want to learn more about it, here’s some resources for you to learn in the way you prefer:

### Getting started with Hilt

Learn how to add Hilt in your Android app with this [guide](https://developer.android.com/training/dependency-injection/hilt-android#setup).

### Documentation

If you’re new to DI or Dagger, check out our [guide to add Hilt to an Android app](https://developer.android.com/training/dependency-injection/hilt-android). Alternatively, if you already know Dagger, we’re also providing [documentation on dagger.dev](https://dagger.dev/hilt).

If you’re just curious about the new annotations and what you can do with Hilt, check out this cheat sheet in the section below.

### For Dagger users

If you’re already using Dagger or dagger.android in your app, check out this [migration guide](https://dagger.dev/hilt/migration-guide) or the codelab mentioned below to help you switch to Hilt. As Dagger and Hilt can coexist together, you can migrate your app incrementally.

### Codelabs

To learn Hilt in a step-by-step approach, we just released two codelabs:

* [Using Hilt in your Android app](https://codelabs.developers.google.com/codelabs/android-hilt)

* [Migrate from Dagger to Hilt](https://codelabs.developers.google.com/codelabs/android-dagger-to-hilt)

### Sample code

Do you want to see how Hilt is used in existing apps? Go check its usage in the [Google I/O 2020 app](https://github.com/google/iosched) and in the [`dev-hilt`](https://github.com/android/architecture-samples/tree/dev-hilt) branch of the Android [architecture-samples Github repository](https://github.com/android/architecture-samples/tree/dev-hilt).

## Feedback

Hilt is currently in *alpha*, try it in your app and give us feedback using this [link](https://github.com/google/dagger/issues/new).

## Cheat sheet

This cheat sheet allows you to quickly see ***what*** the different Hilt and Dagger annotations do and ***how*** to use them.

[**Download cheat sheet in PDF**](https://developer.android.com/images/training/dependency-injection/hilt-annotations.pdf)

![img](assets/images/2020-06-10-di-with-hilt.png)
<small>Hilt and Dagger annotations cheat sheet. [Download in PDF here]((https://developer.android.com/images/training/dependency-injection/hilt-annotations.pdf)).</small>