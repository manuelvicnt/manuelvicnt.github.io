---
layout: post
current: post
cover: assets/images/2019-12-04-Coroutines-gotta-catch-em-all.png
navigation: True
title: Coroutines! Gotta catch 'em all! - KotlinConf 2019
date: 2019-12-04 00:00:00
tags: [talks, coroutines]
class: post-template
subclass: 'post'
author: manuel
---

All about cancellation and exceptions in coroutines

### Abstract

You've added coroutines to your app and everything is fine while your users are on the happy path. But what happens if you cancel a coroutine, you get a timeout or other type of error? Where do you handle it?

Kotlin added structured concurrency to scope the lifetime of a coroutine. But what kind of scopes should you use? How do scopes affect error handling? Does the way you start a coroutine, using launch or async have any effect on the type of errors or the way you handle them?

In this talk we'll go over all of these use cases and show how they can be implemented to ensure robust error handling and a good user experience, even when you're thrown off the happy path.

<iframe width="560" height="315" src="https://www.youtube.com/embed/w0kfnydnFWI" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

### Presenters

[Florina](https://twitter.com/FMuntenescu) is working as an Android Developer Advocate at Google, helping developers architect clean, testable apps using the Architecture Components libraries. She has been working with Android for 8 years, previous work covering news at upday, payment solutions at payleven and navigation services at Garmin.

[Manuel](https://twitter.com/manuelvicnt) is an Android Developer Relations Engineer at Google. With previous experience at Capital One, he currently focuses on App Architecture, Kotlin & Coroutines, Dependency Injection and Jetpack Compose.
