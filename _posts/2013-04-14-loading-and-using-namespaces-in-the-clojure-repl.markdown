---
title: Loading and using namespaces in the Clojure REPL
---
Here's a handy little guide to using your Clojure namespaces in the REPL because it is not immediately obvious how to do this (or at least it was not for me). This assumes you're in the repl started by running `lein repl` in your project root. First, you need to load the namespace using the `load` function passing it the namespace but as a string and as it would look as a file path without the filename extension (i.e., underscores instead of dashes, slashes instead of dots). Second, you can switch to the namespace by using the `in-ns` function and passing it a quoted namespace as an argument.

{% highlight clojure %}
(load "path to namespace")
(in-ns 'namespace)
{% endhighlight %}

You should now have access to all symbols in that namespace and you'll have access to its dependencies with the referred symbols and aliases as defined in the `ns` declaration of the namespace you've switched into. Unfortunately, running `in-ns` without running `load` first will create the namespace as a new empty one in the REPL instead of using the definition in your source code.

As an example with a file located at `<project_root>/src/clj_todo_list/handler.clj` with the following namespace declaration:

{% highlight clojure %}
(ns clj-todo-list.handler
  (:use compojure.core)
  (:require [compojure.handler :as handler]
            [compojure.route :as route]
            [clojure.data.json :as json]))
{% endhighlight %}

Here's how to use the REPL to test some code in that handler file (in this case, reading in a JSON string):

{% highlight clojure %}
user=> (load "clj_todo_list/handler")
com.mchange.v2.log.MLog <clinit>
INFO: MLog clients using java 1.4+ standard logging.
nil
user=> (in-ns 'clj-todo-list.handler)
#<Namespace clj-todo-list.handler>
clj-todo-list.handler=> (json/read-str "{\"content\": \"test\"}")
{"content" "test"}
{% endhighlight %}
