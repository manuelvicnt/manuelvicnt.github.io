---
layout: post
current: post
cover: assets/images/2020-07-01-dagger-hilt-navigation-android-studio.png
navigation: True
title: Dagger and Hilt navigation support in Android Studio
date: 2020-07-01 00:00:00
tags: [hilt, dagger]
class: post-template
subclass: 'post'
author: manuel
---

Easily navigate between Dagger and Hilt related code in Android Studio

> Last update: July 23rd, 2020

Have you ever got lost in a project trying to figure out from where a Dagger or Hilt dependency is being provided? Does it come from an `@Inject` constructor? Or maybe from an `@Binds` or `@Provides` method? Does it have a qualifier? It’s not an easy task…

🤔 What if you could know that and more with just one click? 🎯 Ask no more!

Android Studio 4.1 comes with **new gutter icons that allows you to easily navigate between Dagger-related code**: dependency producers and consumers, components, subcomponents, and modules! Also, you can find the same information in ***Find usages***.

**Hilt support** was added to Android Studio 4.2. Apart from the Dagger features listed above, you can also benefit from easy navigation for **entry points**.

![img](assets/images/2020-04-23-dagger-hilt-navigation-android-studio_1.gif)
<small>Easy Dagger and Hilt dependency graph navigation in Android Studio</small>

As you can see, navigating the Dagger graph of your Android app has never been easier! Knowing from *exactly* which provider method a dependency is coming is just **one click away** with the new support in Android Studio.

## In action

Starting with Android Studio 4.1 Canary 7, you can see a new gutter icon in projects that use Dagger or Hilt:

<p align="center">
	<img height="75" src="assets/images/2020-04-23-dagger-hilt-navigation-android-studio_2.png">
	<small>New Dagger and Hilt gutter icons in Android Studio</small>
</p>

The behavior of these actions are as follows:

* Icon with arrow up -> where the type is provided (i.e. where dependencies come from).

* Tree-shaped icon -> where the type is used as a dependency.

Let’s see some examples of the new functionality using the Dagger branch ([`dev-dagger`](https://github.com/android/architecture-samples/tree/dev-dagger)) of the [architecture-samples GitHub](https://github.com/android/architecture-samples/tree/dev-dagger) sample.

### Knowing where dependencies are coming from

Given a class that can be injected by Dagger, if you tap in the gutter icon with the arrow up of a dependency, you’ll navigate to the method that tells Dagger how to provide that type.

In the following example, `TasksViewModel` has a dependency on `TasksRepository`. Tapping on the gutter icon takes you to the `@Binds` methods in `AppModuleBinds` that provides `TasksRepository`:

![img](assets/images/2020-04-23-dagger-hilt-navigation-android-studio_3.gif)
<small>Know where a dependency is coming from</small>

### Also works with qualifiers!

Given the above, if the dependency is provided using a qualifier, it will take you to exactly that provider method!

`DefaultTasksRepository` depends on a `TasksDataSource` provided with a qualifier. Tapping on the gutter icon takes you to the method in `AppModule` that provides that type with that qualifier:

![img](assets/images/2020-04-23-dagger-hilt-navigation-android-studio_4.gif)
<small>It also works with qualifiers!</small>

### Where is this type being used as a dependency?

When you have a method that tells Dagger how to provide a dependency, you can click the gutter icon with the arrow down to navigate to where that dependency is used. If that dependency is used by more than one consumer, you can select the consumer you want to navigate to from a list.

In our project, `DefaultTasksRepository` is used by different ViewModels. Which ones? You can know it by tapping on the gutter icon of the provider method (`@Binds` in this case):

![img](assets/images/2020-04-23-dagger-hilt-navigation-android-studio_5.gif)
<small>Know where a type is used as a dependency</small>

### Hilt entry points

When you’re at a [Hilt entry point](https://developer.android.com/training/dependency-injection/hilt-android#not-supported), the gutter action helps you navigate to where a dependency is coming from. To showcase this feature, we’ll use the [`interop`](https://github.com/googlecodelabs/android-dagger-to-hilt/tree/interop) branch of the [migrating Dagger to Hilt codelab](https://codelabs.developers.google.com/codelabs/android-dagger-to-hilt/).

![img](assets/images/2020-04-23-dagger-hilt-navigation-android-studio_6.gif)
<small>Navigate where a type comes from at an entry point</small>

## Find usages

You can find the same relationships between your Dagger/Hilt code with the *Find usages* feature in Android Studio.

If you right-click on the `bindRepository` of the `AppModuleBinds` class and select ***Find usages***, for example, you’ll see something similar to this:

![img](assets/images/2020-04-23-dagger-hilt-navigation-android-studio_7.png)
<small>Find usages about bindRepository</small>

---

What are you waiting for to give it a try? Please, use it in your projects and give us feedback in this [link](https://issuetracker.google.com/issues/new?component=192708&template=840533&title=%5BPlease+title+your+report%5D+%23dagger-support). Hope you enjoy it!