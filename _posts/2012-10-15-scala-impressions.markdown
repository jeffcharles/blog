---
title: Scala Impressions
---
Recently I've been playing around with version 2.9.2 of the [Scala programming language](http://www.scala-lang.org/). I used it at work for some basic XML manipulation and at home for a [graphical HTTP client](https://github.com/jeffcharles/GraphicalHttpClient). I am happy with the language but have mixed feelings about its standard library and ecosystem.

## The Core Language

Scala has some amazing resources for learning the language. Personally, I purchased [Programming in Scala (2nd edition)](http://www.artima.com/shop/programming_in_scala_2ed) and was very happy with it as it covered most  of the language features in depth with an explanation of how they are intended to be used. I highly recommend it if you intend to really learn Scala.

The language itself is static and object-oriented with some functional features. I find the core language syntax to be very similar to C#'s with some features from Haskell included. Things I like which I don't see in traditional languages are:

* [Pattern matching](http://www.artima.com/scalazine/articles/pattern_matching.html) which enables all sorts of interesting things like optional return types (similar to using output parameters with bool or enum return types but better).
* Extensive type inference. The only types you need to explicitly state are function parameter types (though it usually makes sense to explicitly state others).
* The val keyword which is similar to the var keyword in C# except you cannot re-assign a val (similar to final in Java).
* Light-weight type definitions.
* Structural type constraints. An example being a generic method that takes a type which implements a parameter-less close method with a unit (void in traditional parlance) return type to simulate C#'s using statement.
* The fact that Call-By-Name function parameters, multiple sets of arguments, and special syntax allowing curly braces instead of parentheses when one argument is in the set allows developers to extend the language [Akka](http://akka.io/) is a Scala library that takes advantage of this).
* [Stackable modifications](http://www.artima.com/scalazine/articles/stackable_trait_pattern.html) which are like the decorator pattern on steroids.

Scala's core language features impress me and I wish the developers behind C# and Java could take a look at incorporating some of them into their languages.

There's a number of high profile people on the Internet claiming that Scala is a complicated language that's difficult to comprehend. Personally I haven't found that to be the case, I find the features useful for the most part and not too hard to figure out after reading through Programming in Scala.

## The Standard Library

The quality of a standard library can make or break first impressions of a language. I think Scala falls hard on this point. The main problems are that it is inconsistent, poorly documented, over-uses operator overloading, relies somewhat on Java's notoriously terribly designed standard library, and in some cases has weird bugs.

The inconsistent and poorly documented aspects caught up with me when using Scala's Swing API. I wanted to create a layer that offered something similar to WPF's (Windows Presentation Foundation) data binding. While I was mostly successful (which I think illustrates the power of the core language), I ran into an odd bug. My GUI uses a TextField and two TextAreas for inputs and I wired up reactions to propogate their values into the backing model when the EditDone event was fired. What I didn't realize was that EditDone only fires for TextFields and not TextAreas. There wasn't really anything documenting this and in fact, there's [a bug filed about it](https://issues.scala-lang.org/browse/SI-5492). To prevent this bug from affecting API consumers, I would've created a trait to mark classes that do publish EditDone and then made sure the EditDone event constructor required a component that extends that trait so API consumers would get a compile time error if they tried doing what I did.

I feel that the Scala standard library over-uses operator overloading. I really like that the core language allows operator overloading since it can make obvious operations more succinct and less-error prone (e.g., using == instead of equals). However, operator overloading is a bad thing when it is used to overload operators that have no clear meaning. I have no idea what ++= is supposed to do on a collection without looking it up and hoping someone bothered to document it.

The scala.actors.Futures seems to be a little broken in 2.9.2. Calling respond or foreach on a Future triggers a `java.lang.interruptedexception`, in another thread with a stacktrace consisting entirely of internal Scala classes, which seems spurious as far as I can tell and can't be swallowed. As well, it seems that running the same block of code again results in the delegate in the future running inside the same thread instead of in another thread. I'm guessing it's all related to [this bug](https://issues.scala-lang.org/browse/SI-5574). Things like this really should be tested before a new version of the standard library ships.

My main gripe with the Java point has to do with me having to use the `HttpURLConnection` class from the Java standard library. I don't blame Scala for this other than that the Scala community markets using or wrapping the Java API as a way of dealing with missing functionality in the standard library. The problems with the `HttpURLConnection` break down to the following:

* The request method taking a string representing the verb rather than an enumeration (or offering an overload that takes an enumeration). There are only seven legal verbs so it doesn't make sense to open up the possibility for a typo.
* The class represents both the request and response which mean that many of the methods offered will throw exceptions if you call them when the object is in the wrong state (e.g., can't set the request method after the connection is established, can't read the input stream (the response stream) until after the connection is established). It's not too hard to avoid hitting these exceptions but, in my view, it defeats one of the main advantages of using a static language when you offer an API that allows a clearly invalid program to compile but always fail at runtime.
* You need to call `setDoOutput(true)` to write into the output stream (the request stream). This seems like something the implementation could do for me. If I open the input stream, I am clearly intending to write to it.
* There are two different timeouts involved, connection and read. To me, the amount of time involved reading is probably going to be the vast majority of time involved so there's not much point offering both timeouts. It just adds noise and confusion

There are other issues with this class but these are the main ones that annoy me.

## The ecosystem

Scala has access to most of Java's ecosystem and a few of its own tools. I was very pleased to see that Eclipse, IntelliJ IDEA, and Netbeans all have Scala plugins. It's nice to have a number of free options as opposed to C# where you have to choose between a limited free version of Visual Studio, a pricey version of Visual Studio, or MonoDevelop (which I wasn't happy with the last time I tried it). I did run into trouble with using Eclipse when upgrading the JDT (Java Development Tools) broke the Scala plugin without warning. Usually when a piece of software has automatic dependency management, I assume it will warn me when upgrading one component will break another. That being said, it may have an issue with the Scala plugin not reporting its dependencies correctly. However, as I found a bug filed around the PyDev plugin complaining about similar breakage, I'm assuming it's a problem with Eclipse. I switched over to IntelliJ which seemed to work fine with the exception of it not placing the Java standard library in the classpath. I ended up getting around that by just using SBT to compile and run the project. I'm sure there's a fix that would allow IntelliJ to build the project, I just didn't feel like looking into it.

Scala, similar to Java, has two mainstream build tools, Maven and SBT (Simple Build Tool). Initially, I was just using whatever Eclipse uses under the covers to build the project, but when Eclipse broke I needed a build tool. Despite my previous positive experience with Maven, I decided to give SBT a try because I like learning new things. SBT's 14 page getting started guide is a little intimidating, especially considering SBT markets itself as a "simple" tool. In reality, I only needed to read the first five or so pages to get everything set up properly. The rest probably should not be in the getting started guide. This relatively simple [commit](https://github.com/jeffcharles/GraphicalHttpClient/commit/42e2aa687b898567c8b4b0b45cc8d024d1451806) was all that was necessary. One thing that does strike me though, is that for a simple project like this, Maven's single `pom.xml` file is even simpler. I'm not really sold on the benefits SBT brings, in particular, I really wish they had stuck with valid Scala for the build definition rather than invent their own slightly finicky domain specific language.

## Conclusion

Despite my misgivings, I believe Scala represents a step in the right direction. I believe the core language offers clear benefits over Java and C# as languages. If the choice is between Scala and Java, I'd pick Scala since it's a safer and more expressive language. Ultimately, you have the option of not using its standard library and falling back on the Java one which is what you would have to do anyway if you were to use Java. The .NET framework offers a very compelling standard library so the choice is much more difficult between C# and Scala and I think a lot of that decision would have to do with the particulars of the project. For scripts and minor applications, I think Python is a better choice because of its superior standard library.
