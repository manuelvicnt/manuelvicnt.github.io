---
layout: post
current: post
cover: assets/images/2019-07-30-dagger-kotlin-optimizations.png
navigation: True
title: Dagger in Kotlin - Gotchas and Optimizations
date: 2019-07-30 00:00:00
tags: [dagger]
class: post-template
subclass: 'post'
author: manuel
---

Dagger optimizations come with no cost! Add them and follow best practices

[Dagger](https://dagger.dev/) is a popular Dependency Injection framework commonly used in Android. It provides fully static and compile-time dependencies addressing many of the development and performance issues that have reflection-based solutions.

This month, a [new tutorial](https://dagger.dev/tutorial/) was released to help you better understand how it works. This article focuses on using **Dagger with Kotlin**, including **best practices** to optimize your build time and **gotchas** you might encounter.

Dagger is implemented using Java’s annotations model and annotations in Kotlin are not always directly parallel with how equivalent Java code would be written. This post will highlight areas where they differ and how you can use Dagger with Kotlin without having a headache.

This post was inspired by some of the suggestions in this [Dagger issue](https://github.com/google/dagger/issues/900) that goes through best practices and pain points of Dagger in Kotlin. Thanks to all of the contributors that commented there!

## kapt build improvements

To improve your build time, Dagger added support for **gradle’s incremental annotation processing** in [v2.18](https://github.com/google/dagger/releases/tag/dagger-2.18)! This is enabled by default in Dagger [v2.24](https://github.com/google/dagger/releases/tag/dagger-2.24). In case you’re using a lower version, you need to add a few lines of code (as shown below) if you want to benefit from it.

Also, you can tell Dagger not to format the generated code. This option was added in Dagger [v2.18](https://github.com/google/dagger/releases/tag/dagger-2.18) and it’s the default behavior (doesn’t generate formatted code) in [v2.23](https://github.com/google/dagger/releases/tag/dagger-2.23). If you’re using a lower version, disable code formatting to improve your build time (see code below).

Include these compiler arguments in your `build.gradle` file to make Dagger more performant at build time:

```groovy
allprojects {
    afterEvaluate {
        extensions.findByName('kapt')?.arguments {
            arg("dagger.formatGeneratedSource", "disabled")
            arg("dagger.gradle.incremental", "enabled")
        }
    }
}
```

Alternatively, if you use Kotlin DSL script files, include them like this in the `build.gradle.kts` file of the modules that use Dagger:

```groovy
kapt {
    arguments {
        arg("dagger.formatGeneratedSource", "disabled")
        arg("dagger.gradle.incremental", "enabled")
    }
}
```

## Qualifiers for field attributes

When an annotation is placed on a property in Kotlin, it’s not clear whether Java will see that annotation on the *field* of the property or the method for that property. Setting the `field:` prefix on the annotation ensures that the qualifier ends up in the right place ([see documentation](https://kotlinlang.org/docs/reference/annotations.html#annotation-use-site-targets) for more details).

✅ The way to apply qualifiers on an injected field is:

```kotlin
@Inject @field:MinimumBalance lateinit var minimumBalance: BigDecimal
```

❌ As opposed to:

```kotlin
@Inject @MinimumBalance lateinit var minimumBalance: BigDecimal 
// @MinimumBalance is ignored!
```

Forgetting to add `field:` could lead to injecting the wrong object if there’s an unqualified instance of that type available in the Dagger graph.

**Update 10/28/19**: This was fixed in Dagger v2.25. If you use this version, you can just type what wasn’t working before:

```kotlin
@Inject @MinimumBalance lateinit var minimumBalance: BigDecimal 
// FIXED: @MinimumBalance is NOT ignored!
```

## Static @Provides functions optimization

Dagger’s generated code will be more performant if `@Provides` methods are `static`. To achieve this in Kotlin, use a Kotlin `object` instead of a `class` and annotate your methods with `@JvmStatic`. This is a ***best practice*** that you should follow as much as possible.

```kotlin
@Module
object NetworkModule {

    @JvmStatic
    @Provides
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder().build()
    }
}
```

In case you need an abstract method, you’ll need to add the `@JvmStatic` method to a companion object and annotate it with `@Module` too.

```kotlin
@Module
abstract class NetworkModule {

    @Binds abstract fun provideService(retrofitService: RetrofitService): Service

    @Module
    companion object {

        @JvmStatic
        @Provides
        fun provideOkHttpClient(): OkHttpClient {
            return return OkHttpClient.Builder().build()
        }
    }
}
```

Alternatively, you can extract the object module out and include it in the abstract one:

```kotlin
@Module(includes = [OkHttpClientModule::java])
abstract class NetworkModule {
    
    @Binds abstract fun provideService(retrofitService: RetrofitService): Service
}

@Module
object OkHttpClientModule {

    @JvmStatic
    @Provides
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder().build()
    }
}
```

**Update 10/28/19**: With Dagger v2.25.2, you don’t need to tag the `@Provides` function with `@JvmStatic`. Dagger will understand it properly.

## Injecting Generics

Kotlin compiles generics with wildcards to make Kotlin APIs work with Java. These are generated when a type appears as a parameter ([more info here](https://kotlinlang.org/docs/reference/java-to-kotlin-interop.html#variant-generics)) or as fields. For example, a Kotlin `List<Foo>` parameter shows up as `List<? super Foo>` in Java.

This causes problems with Dagger because it expects an exact (aka [invariant](https://en.wikipedia.org/wiki/Class_invariant)) type match. Using [`@JvmSuppressWildcards`](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.jvm/-jvm-suppress-wildcards/index.html) will ensure that Dagger sees the type without wildcards.

This is a common issue when you inject collections using [Dagger’s multibinding feature](https://dagger.dev/multibindings.html), for example:

```kotlin
class MyVMFactory @Inject constructor(
  private val vmMap: Map<String, @JvmSuppressWildcards Provider<ViewModel>>
) { 
    /* ... */
}
```

## Inline method bodies

Dagger determines the types that are configured by `@Provides` methods by inspecting the return type. Specifying the return type in Kotlin functions is optional and even the IDE sometimes encourages you to refactor your code to have inline method bodies that hide the return type declaration.

This can lead to bugs if the inferred type is different from the one you meant. Let’s see some examples:

If you want to add a specific type to the graph, inlining works as expected. See the different ways to do the same in Kotlin:

```kotlin
@Provides 
fun provideNetworkPrinter() = NetworkPrinter()

@Provides 
fun provideNetworkPrinter(): NetworkPrinter = NetworkPrinter()

@Provides 
fun provideNetworkPrinter(): NetworkPrinter {
  return NetworkPrinter()
}
```

If you want to provide an implementation of an interface, then you must explicitly specify the return type. Not doing it can lead to problems and bugs:

```kotlin
@Provides
// configures a `Printer`
fun providePrinter(): Printer = NetworkPrinter() 

@Provides
// configures a `NetworkPrinter`, not a plain `Printer`!
fun providePrinter() = NetworkPrinter() 
```

---

Dagger mostly works with Kotlin out of the box. However, you have to watch out for a few things just to make sure you’re doing what you really mean to do: `@field:` for qualifiers on field attributes, inline method bodies, and `@JvmSuppressWildcards` when injecting collections.

Dagger optimizations come with no cost, add them and follow best practices to improve your build time: enabling incremental annotation processing, disabling formatting and using static `@Provides` methods in your Dagger modules.