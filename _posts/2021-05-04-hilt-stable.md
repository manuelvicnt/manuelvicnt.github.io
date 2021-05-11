---
layout: post
current: post
cover: assets/images/2021-05-04-hilt-stable.png
navigation: True
title: Hilt is stable! Easier dependency injection on Android
date: 2021-05-04 00:00:00
tags: [hilt]
class: post-template
subclass: 'post'
author: manuel
---

Hilt, Jetpack’s recommended dependency injection (DI) solution for Android apps, is already stable!

[Hilt](https://developer.android.com/training/dependency-injection/hilt-android), Jetpack’s recommended [dependency injection (DI)](https://developer.android.com/training/dependency-injection) solution for Android apps, is already **stable**! That means Hilt is fully ready to be used **in production**. Hilt is simpler than Dagger, enables you to write less boilerplate code, it’s designed for Android and has integration with multiple Jetpack libraries. Several companies have already started taking advantage of Hilt in their apps.

Hilt was [first released](https://manuelvivo.dev/di-with-hilt) as alpha in June 2020 with the mission of defining a **standard way** to do DI in your Android apps and since then, we’ve received tons of feedback from developers. That not only improved the library, but also, it let us know that we’re working on the right problems.

Instead of creating dependency graphs by hand, and manually injecting and passing around types where needed, Hilt automatically generates all that code for you at compile time by means of annotations. Hilt can help you **get the most out of DI best practices** in your app by doing the hard work and **generating all that boilerplate** you would’ve needed to write otherwise. Also, as it’s fully integrated with Android, Hilt manages the lifecycle of the dependency graphs associated with the Android framework classes automatically for you.

Let’s see Hilt in action with a quick example! After [setting Hilt up](https://developer.android.com/training/dependency-injection/hilt-android#setup), using it in your project from scratch to inject a ViewModel in an Activity is as easy as adding few annotations to your code as follows:

```kotlin
@HiltAndroidApp // Setup Hilt in your app
class MyApplication : Application() { ... }

// Make Hilt aware of this ViewModel
@HiltViewModel
class LoginViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    /* ... Other dependencies Hilt takes care of ... */
) : ViewModel() { ... }


// Make the activity use the right ViewModel factory and
// inject other dependencies
@AndroidEntryPoint 
class LoginActivity : AppCompatActivity() {

    private val loginViewModel: LoginViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // loginViewModel is ready to be used
    }
}
```

Apart from what’s mentioned above, why should you use Hilt in your Android app?

## Simpler than Dagger

Hilt is built on top of the popular DI library [Dagger](https://developer.android.com/training/dependency-injection/dagger-basics) so benefits from the compile time correctness, runtime performance, scalability, and [Android Studio support](https://manuelvivo.dev/dagger-hilt-navigation-android-studio) that Dagger provides. Some Dagger annotations, such as @Inject to tell Dagger and Hilt how to provide an instance of a type, are often used in Hilt. But Hilt is simpler than Dagger!

> *"I highly recommend leveraging Dagger for dependency injection in Android applications. However, pure vanilla Dagger can lead to too much room for creativity. When that gets mixed with the complexity of the various lifecycle-aware components that are part of Android development, there’s plenty of room for pitfalls such as memory leaks: for example, accidentally passing in Activity-scoped dependencies into ViewModels. Hilt being opinionated and designed specifically for Android helps you avoid some of the pitfalls when using vanilla Dagger."* — [Marcelo Hernandez](https://twitter.com/mhernand40), Staff Software Engineer, Tinder

If you’re already using Dagger in your app and want to migrate to Hilt… fear not! Dagger and Hilt can coexist together, apps can be [migrated](https://dagger.dev/hilt/migration-guide) on an as-needed basis.

## Less boilerplate

Hilt is opinionated — this means that it makes decisions for you so that you have less code to write. Hilt defines standard components, or dependency graphs, fully integrated with Android framework classes such as Application, activities, fragments, and views. As well as scope annotations to scope instances of types to those components.

> *"Hilt automatically generates the test application and test component via @HiltAndroidTest. We were able to remove between 20% and 40% of boilerplate wire up test code after migrating to Hilt!"* — Jusun Lee, Software Engineer, YouTube

> *"We are only scratching the surface in terms of migrating to Hilt. However, in one of the modules we migrated to Hilt, we saw +78/-182 in terms of lines changed for this library."* — Marcelo Hernandez, Staff Software Engineer, Tinder

## Designed for Android

As opposed to Dagger that is a dependency injection solution for the Java programming language applications, Hilt solely works in Android apps. Some annotations such as `@HiltAndroidApp`, `@AndroidEntryPoint`, or `@HiltViewModel` are specific to Hilt and define an opinionated way to do DI on Android.

> *"Hilt finally offers built-in Android lifecycle aware Dagger components. With Hilt, we can solely focus on Dagger @Modules without having to worry about Components, Subcomponents, the Component provider pattern, and so on."* — Marcelo Hernandez, Staff Software Engineer, Tinder

## Standardization of components and bindings

For those that know about Dagger, Hilt simplifies the dependency graph by using a [monolithic component system](https://dagger.dev/hilt/monolithic) to generate less code at compile time.

> *"With Hilt’s monolithic component system, binding definitions are provided once and shared across all classes that use that component. This is a big win as previously, YouTube used a polylithic component system where modules were manually wired-up into custom components and there were many duplicate binding definitions."* — Jusun Lee, Software Engineer, YouTube

> *"As our Gradle modules separation allows for feature development in isolation, it became easy to be too creative when using Dagger. We’ve found that migrating those modules over to Hilt has actually exposed flaws in which we were inadvertently violating the separation of concerns."* — Marcelo Hernandez, Staff Software Engineer, Tinder

## Integration with other Jetpack libraries

You can use your favourite Jetpack libraries with Hilt out of the box. We provide direct injection support for **ViewModel, WorkManager, Navigation, and Compose** so far.

Learn more about Jetpack support in the [docs](https://developer.android.com/training/dependency-injection/hilt-jetpack).

> *"I really appreciate how Hilt works out of the box with ViewModels and how it eliminates the boilerplate of having to set up a ViewModel.Factory provider with vanilla Dagger."* — Marcelo Hernandez, Staff Software Engineer, Tinder

## Resources to learn Hilt

Hilt is Jetpack’s recommended DI solution for Android apps. To learn more about it and start using it in your apps, check out these resources:

* Learn about the benefits of dependency injection [here](https://developer.android.com/training/dependency-injection).
* [Documentation](https://developer.android.com/training/dependency-injection/hilt-android) to learn how to use Hilt in your app.
* [Migration guide](https://dagger.dev/hilt/migration-guide) from Dagger to Hilt.
* Codelabs to learn Hilt in a step-by-step approach: [Using Hilt in your Android app](https://codelabs.developers.google.com/codelabs/android-hilt) and [Migrating from Dagger to Hilt](https://codelabs.developers.google.com/codelabs/android-dagger-to-hilt).
* Code samples! Check out Hilt in action in the [Google I/O 2020](https://github.com/google/iosched) and [Sunflower](https://github.com/android/sunflower/) apps.
* [Cheat sheet](https://developer.android.com/images/training/dependency-injection/hilt-annotations.pdf) to quickly see *what* the different Hilt and Dagger annotations do and *how* to use them.

---

Huge thanks to [Marcelo Hernandez](https://twitter.com/mhernand40) from the Android Tinder team, and Jusun Lee from the Android YouTube team for taking the time to talk about how and why they’re adopting Hilt in their apps.
