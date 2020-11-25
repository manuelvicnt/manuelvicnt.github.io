---
layout: post
current: post
cover: assets/images/2020-11-24-migrating-to-Hilt.png
navigation: True
title: Migrating from Dagger to Hilt — Is it worth it?
date: 2020-11-24 00:00:01
tags: [hilt]
class: post-template
subclass: 'post'
author: manuel
---

Consider whether or not you should migrate your Dagger app to Hilt

Hilt got released in June 2020 as a way to standardize dependency injection (DI) in Android. For new projects, Hilt provides compile time correctness, runtime performance and scalability (read more about that [here](https://manuelvivo.dev/di-with-hilt))! However, what are the benefits for an application that already uses Dagger? **Should you be migrating** your current app to Hilt? The following are some reasons whether your team should invest migrating from Dagger to Hilt.

## ✅ AndroidX extensions

If you already have Dagger working with ViewModels or WorkManager, you saw that wiring up your ViewModelFactory and WorkerFactory requires quite a lot of boilerplate code and Dagger knowledge. The most common implementation uses [multibindings](https://dagger.dev/dev-guide/multibindings.html) which is one of the most complex features in Dagger that developers often struggle to understand. Hilt makes working with AndroidX a lot easier by removing that boilerplate code. What’s even better is that you don’t even need to inject the Factory in the Android framework class, you call it as if Hilt wasn’t there. With `@ViewModelInject`, Hilt generates the right `ViewModelProvider.Factory` for you that `@AndroidEntryPoint` activities and fragments can use directly.

```kotlin
class PlayViewModel @ViewModelInject constructor(
  val db: MusicDatabase,
) : ViewModel() {/* ... */}

@AndroidEntryPoint
class PlayActivity : AppCompatActivity() {

  val viewModel: PlayViewModel by viewModels()

  override fun onCreate(savedInstanceState: Bundle) {
    super.onCreate(bundle)
    viewModel.play()
  }
}
```

## ✅ Testing APIs

DI is supposed to make testing easier but ironically, having Dagger working in tests requires [a lot of work](https://developer.android.com/training/dependency-injection/dagger-android#dagger-testing). The fact that you have to maintain both the prod and test Dagger graph at the same time makes it notably worse than [Hilt’s approach](https://developer.android.com/training/dependency-injection/hilt-testing).

Hilt tests can explicitly modify the DI graph using the [`@UninstallModules`](https://developer.android.com/training/dependency-injection/hilt-testing#replace-binding) functionality. Apart from that, you get other perks like [`@BindValue`](https://developer.android.com/training/dependency-injection/hilt-testing#binding-new) that allows you to easily bind fields of your tests into the DI graph.

```kotlin
@UninstallModules(AnalyticsModule::class)
@HiltAndroidTest
class ExampleTest {
  
  @get:Rule
  var hiltRule = HiltAndroidRule(this)
  
  @BindValue @JvmField
  val analyticsRepository = FakeAnalyticsRepository()
  
  @Test 
  fun myTest() { ... }
}
```

## ✅ Consistency

There are multiple ways to have the same functionality working in Dagger. The historical lack of guidance for Android apps (that we [tackled](https://developer.android.com/training/dependency-injection/dagger-basics) last year) has caused multiple debates in the community and ultimately created inconsistencies in the way developers use and set up Dagger in their Android apps.

You might argue that your current Dagger setup is really good and you perfectly know how everything works and how everything is getting injected. Therefore, migrating to Hilt is not worth it! That might be true in your case, but is it the same for the rest of the team (and potentially future colleagues)? Will you know how everything works when switching to a new project? Understanding the setup and usage of Dagger in an app can be challenging and time consuming.

That time can be dramatically reduced by using Hilt into your app as the same setup is used by all Hilt applications. A new developer joining your team won’t be surprised about your Hilt setup because it’ll be pretty much the same as what they’re used to.

## ✅ Custom Components

Apart from the defined standard components, Hilt also gives you a way to create custom components and add them to the hierarchy which you can read more about [here](https://manuelvivo.dev/hilt-adding-components).

Even though custom components reduce consistency, you still get a lot of benefits! The module auto-discoverability (i.e. the [`@InstallIn`](https://developer.android.com/training/dependency-injection/hilt-android#hilt-modules) annotation functionality) feature as well as the test replacement features also work with custom components.

However, the difference between custom components and the Hilt built-in components is that you lose the ability to automatically inject those components into Android framework classes (i.e. what `@AndroidEntryPoint` does).

## ✅ Dagger and Hilt interop

Hilt and Dagger can co-exist together! You can benefit from Hilt in certain parts of your app while keeping the other most niche parts using Dagger if you allow Hilt to take over your `SingletonComponent`. This also means that the [migration to Hilt can be done gradually](https://codelabs.developers.google.com/codelabs/android-dagger-to-hilt#0).

## ❌ Component dependencies

Hilt being opinionated means it makes decisions for you. Hilt uses subcomponents for the component relationships, ready why [here](https://dagger.dev/hilt/monolithic). If you’re a strong believer that your app is better off using component dependencies, Hilt is not the right tool for your app.

---

Migrating from Dagger to Hilt is worth it in most projects. The benefits Hilt brings to your application outnumbers the efforts of having to update. But you are not on your own! We provided lots of resources to help you out in this journey:

* Comprehensive migration [documentation](https://dagger.dev/hilt/migration-guide).

* Migrating from Dagger to Hilt [codelab](https://codelabs.developers.google.com/codelabs/android-dagger-to-hilt#0).

* Migrating the Google I/O app to Hilt [blog post](https://medium.com/androiddevelopers/migrating-the-google-i-o-app-to-hilt-f3edf03affe5) and [commit](https://github.com/google/iosched/commit/9c20fdd52d446e5fdb03369e50fb196c31ae16e3).

* Hilt and Assisted Injection working together [gist](https://gist.github.com/manuelvicnt/437668cda3a891d347e134b1de29aee1).

Leave a comment below if you have any questions or you’re missing any more information about this!
