---
layout: post
current: post
cover: assets/images/2021-12-14-new-app-architecture.png
navigation: True
title: Rebuilding our guide to app architecture
date: 2021-12-14 00:00:00
tags: [architecture]
class: post-template
subclass: 'post'
author: manuel
---

We just launched a revamped guide to app architecture.

## Description 

As Android apps grow in size, it's important to design the code with an architecture in place to allow the app to **scale**, improve **quality** and **robustness**, and make it **easier to test**.

An app architecture defines the **boundaries** between parts of the app and the **responsibilities** each part should have. This favors the [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) principle that enables the aforementioned benefits.

In response to community demand for up-to-date guidance on app architecture, **we're launching a [revamped guide to app architecture](https://developer.android.com/jetpack/guide)**. This includes best practices and recommended architecture for building robust, high-quality apps. It also provides a page for each layer of the recommended architecture: [UI](https://developer.android.com/jetpack/guide/ui-layer), [domain](https://developer.android.com/jetpack/guide/domain-layer), and [data](https://developer.android.com/jetpack/guide/data-layer) layers. Within them, you'll find deep dives into more complex topics, such as how to handle [UI events](https://developer.android.com/jetpack/guide/ui-layer/events).

Each Android app should have at least two layers:

* The [UI layer](https://developer.android.com/jetpack/guide/ui-layer) that displays application data on the screen.

* The [data layer](https://developer.android.com/jetpack/guide/data-layer) that contains the business logic of your app and exposes application data.

You can add an additional layer called the [domain layer](https://developer.android.com/jetpack/guide/domain-layer) to simplify and reuse the interactions between the UI and data layers.

<p align="center">
  <img width="800" src="assets/images/2021-12-14-new-app-architecture_2.png">
  <small>General diagram of a typical app architecture. The UI layer gets the application data from the optional domain layer, or the data layer, that exposes application data.</small>
</p>

We have created a [learning pathway](https://developer.android.com/courses/pathways/android-architecture) to help you consume this content in order and in a trackable way. Don't miss the chance to learn all of this and get a badge as recognition!

<p align="center">
  <img width="600" src="assets/images/2021-12-14-new-app-architecture_3.png">
</p>

### Is this for you?

If you’re a **beginner**, you should begin by **understanding the benefits** of having an app architecture and then follow these recommendations as a first approach to the topic. **Intermediate and advanced** developers can **follow** these recommendations and **customize** them to their needs. In fact, our research suggests that most professional developers are already using these best practices.

You might be wondering if you should update your existing architecture to follow this recommendation, and the answer is no... or wait... it's up to you. If your current architecture works for your team, you might want to stick with it. But you might also find patterns in our guides you can benefit from and incorporate into your app.

### We’re not done yet

This is the first batch of documents we're releasing, with more to come in 2022. Help us make the guidance better! If you have any feedback on the current recommendations or if you want to see other architecture-related topics in them, let us know in our [docs issue tracker](https://issuetracker.google.com/issues/new?component=192697&template=845603).
