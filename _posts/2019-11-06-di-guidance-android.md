---
layout: post
current: post
cover: assets/images/2020-03-30-dagger-cheat-sheets.png
navigation: True
title: Dependency Injection guidance on Android — ADS 2019
date: 2019-11-06 00:00:00
tags: [dagger]
class: post-template
subclass: 'post'
author: manuel
---

Why dependency injection is important on Android and a new guide about it!

> Note: [Hilt](https://manuelvivo.dev/di-with-hilt) is the Jetpack's recommended way for dependency injection on Android.

According to our [introduction to Dependency Injection](https://developer.android.com/training/dependency-injection) (DI), we believe you should always use DI principles in your applications. If you’re developing a professional Android app, use [Dagger](https://dagger.dev/) to better manage your dependencies.

We recommended using Dagger for medium and large apps; for small apps or pet projects, it doesn’t matter the tool you use, but the sooner you add Dagger to your project, the better it will be and the less you will have to refactor in the future.

![img](assets/images/2019-11-06-di-guidance-android_1.png)
<small>DI tool to use depending on the size of your project. This is relative, use your own judgement since every app is different.</small>

If you compare the cost of managing your dependencies when using Dagger vs any other tool visually, Dagger starts with a relatively high cost but plateaus as the app gets bigger.

![img](assets/images/2019-11-06-di-guidance-android_2.png)
<small>Made up graph showing the scalability of different DI techniques</small>

Dagger really shines with:

* **Performance**. Without reflection and the code being generated at build time, Dagger can provide the best in class performance.

* **Correctness**. Dagger gives you security at build time. If a type is not available in the DI application graph, you won’t be able to build the project instead of getting those crashes at runtime.

* **Scalability**. Dagger is built to scale and be suitable for large applications such as Gmail, Google Photos or YouTube.

DI and Dagger are complex topics, for this reason we’re providing documentation and samples to learn these professional tools.

## New Documentation

We just released a set of documentation to help you with this. We’d like to help both beginners and more experienced users by giving good practices.

New documentation available:

* [DI Overview](https://developer.android.com/training/dependency-injection): What’s DI and the benefits of using it in your project.

* [Manual DI](https://developer.android.com/training/dependency-injection/manual): To better understand the benefits of DI and what Dagger can do for you, you should try DI yourself. In this page, we show how you can do DI manually in your app.

* [Dagger basics](https://developer.android.com/training/dependency-injection/dagger-basics): Benefits of adding Dagger to your project and what it does under the hood.

* [Using Dagger in an Android app](https://developer.android.com/training/dependency-injection/dagger-android): Learn all the concepts of Dagger from scratch by adding Dagger to a typical Android app.

* [Dagger in a multi-module application](https://developer.android.com/training/dependency-injection/dagger-multi-module): How to use Dagger in a modularised app with both regular Gradle modules and dynamic feature modules.

## New Codelab

Apart from the documentation, the best way you can learn a topic is with hands on code. We released a new codelab called: [Using Dagger in an Android app](https://codelabs.developers.google.com/codelabs/android-dagger). It starts with a manual DI implementation that you’ll be migrating to Dagger flow by flow of the app.
By the end of it, you’ll build an application graph like this!

![img](assets/images/2019-11-06-di-guidance-android_3.png)
<small>Final Dagger graph built throughout the codelab</small>

Complete the codelab to understand the main concepts of Dagger so you can apply them to your project accordingly.

## Easier Dagger in Android

We want to make Dagger more accessible to small applications by reducing the amount of code you have to write. We’re working on a new initiative that will make Dagger simpler in Android. What will be changing?

* The work you do for Dagger Components can be automated! We’ll have some predefined Components that can simplify the work for you.

* It will be integrated with the rest of Jetpack and will provide an easy integration with ViewModels, WorkManager and Navigation.

* Kotlin-friendly. We want Dagger to work seamlessly when written in Kotlin and expand its capabilities to make it work with delegates and other Kotlin powerful features.

* Dagger Modules are the way to provide information to the Dagger graph. It’s important they stay and serve the same functionality in the new initiative.

Stay tuned, we’re working on that.

## dagger.android

[dagger.android](https://dagger.dev/android) is a library on top of Dagger that reduces boilerplate code when using Dagger in Android framework classes (e.g. Activities or Fragments).

Even though it helped with boilerplate, we think we can do better. Since the new Android approach is radically different, no major improvements are planned for this library but we’re committed to maintaining it until a suitable stable replacement is available.

If you’re using dagger.android, keep using it. If you’re starting a new project, consider dagger.android as an alternative and use if it fits your use case. We’ll provide migration guides from both Dagger and dagger.android code to the new initiative.

## Conclusion

Use Dependency Injection and use Dagger! We’re already working on making it better. For more information, check the Android Dev Summit 2019 recording about this topic:

<iframe width="560" height="315" src="https://www.youtube.com/embed/o-ins1nvbDg" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>