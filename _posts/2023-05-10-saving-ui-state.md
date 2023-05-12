---
layout: post
current: post
cover: assets/images/2023-05-10-saving-ui-state.jpg
navigation: True
title: Saving UI state on Android
date: 2023-05-10 00:00:00
tags: [talks, architecture]
class: post-template
subclass: 'post'
author: manuel
---

Google I/O 2023 talk

## Description 

Saving a UI state properly is essential for providing a great UX experience. Your users might be disappointed if your screen doesn't preserve its state during a configuration change, activity recreation, or system-initiated process death. Learn best practices to save UI state properly in both the View system and Jetpack Compose by comparing APIs such as `remember`, `rememberSaveable`, `onSaveInstanceState`, `ViewModel`, and `SavedStateHandle`.


<iframe width="560" height="315" src="https://www.youtube.com/embed/V-s4z7B_Gnc" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" allowfullscreen></iframe>


### Resources

* [Save UI states](https://developer.android.com/topic/libraries/architecture/saving-states)
* [Save UI state in Compose](https://developer.android.com/jetpack/compose/state-saving)
* [Guide to app architecture](https://goo.gle/mad-architecture-guide)
* [UI layer](https://goo.gle/architecture-ui-layer)
* [State holders and UI State](https://developer.android.com/topic/architecture/ui-layer/stateholders)
* [UI State production](https://developer.android.com/topic/architecture/ui-layer/state-production)
* [ViewModel overview](https://goo.gle/architecture-viewmodel)
* [Architecture recommendations](https://goo.gle/architecture-recommendations)