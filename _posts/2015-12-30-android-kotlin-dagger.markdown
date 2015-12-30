---
title: Android, Kotlin, and Dagger
---

I've recently started work on an Android side-project. As part of this work, I'm experimenting with using Kotlin as a programming language and Dagger as a dependency injection framework.

## Why Kotlin?

Android development is constrained to certain JVM languages. Personally I find Java overly verbose and checked exceptions often make code harder to follow without increasing reliability. Beyond supporting Android as a compilation target, Kotlin has features like ignoring checked exceptions, data classes (value classes for holding immutable data), lazily initialized properties, and null-safety baked into its type system. These features all come together to reduce the amount of code that has to be written and maintained as well as making the code easier to follow.

## Why Dagger?

Dependency injection makes code easier to test by allowing developers to change the behaviour of dependencies during development and testing. This is useful for things like changing return values from a dependency for a given test case and verifying that certain calls with side-effects have occurred. Dagger is a dependency injection framework that uses code annotations to generate the appropriate object wire-up code during compilation instead of using reflection at runtime. This results in less overhead compared to traditional dependency injection frameworks and makes it an appropriate approach for Android.

## Using Kotlin and Dagger together

Getting Kotlin and Dagger to work together is more difficult than it would initially seem. The initial reading I did online about dependency injection on Android led me to Square's Dagger 1 project. Unfortunately, Square's Dagger 1 does not appear to work with Kotlin while Google's Dagger 2 does. Most of the code examples I found used Dagger 1 until I started explicitly searching for Dagger 2. The Java examples don't translate directly either since Kotlin seems to require using `kapt` instead of `apt` or Gradle's `provide`. `kapt` also needs to be configured to generate stubs which is not at all obvious. You also need to include a compile-time library for reading code annotations.

Here is what I put in my `app/gradle.build`:

{% highlight groovy %}
{% raw %}
kapt {
    generateStubs = true
}

dependencies {
    // ... other application dependencies
    kapt 'com.google.dagger:dagger-compiler:2.0'
    compile 'com.google.dagger:dagger:2.0'
    provided 'org.glassfish:javax.annotation:10.0-b28'
}
{% endraw %}
{% endhighlight %}
