---
title: Learning Clojure
---
A few months ago I decided to try to learn Clojure. Clojure is a dynamically-typed functional language using LISP-syntax with compilers available for the JVM (Java Virtual Machine), CLR (Common Language Runtime), and JavaScript. I've gained an appreciation for the language but still have a lot more learning to do before I am proficient in it or other functional languages like Haskell or Scala.

I decided to purchase and read through Clojure Programming by Chas Emerick, Brian Carper, & Christophe Grand and followed that up by starting a spreadsheet application called Transcend. I felt that Clojure Programming provided a very good basis for understanding and writing Clojure. My only previous experience with LISP was reading through the first couple chapters of SICP (Structure and Interpretation of Computer Programs) which used Scheme. When I wrote Transcend, I did not complete enough of it for me to feel that it was a functionally useful project. I feel as though I did not pick up enough of how to design good Clojure APIs after my initial read-through of Clojure Programming. It may be that I should re-read it.

Some people ask me why they should bother learning or using Clojure. The big advantages of the language compared to others I have used are:

* An API design bias toward immutability that performs well
* Better concurrency abstractions (owing in large part to the point above)
* Multiple dynamic dispatch (most other languages I've used just have single dynamic dispatch)
* First class code generation through macros (way better than T4 templates in C#)
* A very simple dynamic type system
* Can compile to Java byte-code or JavaScript

I did run into a few issues while developing Transcend. The big two were debuggability and not using correct design idioms. Since Clojure is a dynamically typed language, all type related errors become run-time errors and the stack traces can get quite deep. There's also no easy way to drop down to a REPL or a similar debugger inside a function when you hit a certain condition or an exception. Most of the debugging suggestions online seem to revolve around adding print statements which can help but a REPL would be a lot nicer and more flexible. I'm guessing there's probably some magic that more experienced Clojure developers have for debugging or avoiding these situations in the first place. I definitely found certain ways of designing interfaces and APIs in the object-oriented world that allowed me to avoid certain hard-to-debug situations.

The design problems in Transcend were due to my lack of understanding of how to write good dynamic functional APIs. I spend most of my time in a static object-oriented world which is almost completely foreign to how Clojure is meant to be used. I found that most of my APIs were written as if I were working in a procedural language rather than a functional one. Again, I'm sure experience and more reading will likely help improve my API designs. Unfortunately, there's less shared experience to be learned from compared to more traditional static object-oriented languages.

My next steps are reading ClojureScript: Up and Running, Clojure in Action, and The Joy of Clojure and perhaps re-reading Clojure Programming. I think JavaScript and HTML5 seem like a better platform for client applications compared to the JVM and Swing. I'm hoping that Clojure in Action and The Joy of Clojure can impart more API design tips.
