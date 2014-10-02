---
title: Better responses for invalid JSON given Ring's wrap-json-body parse issues
---
As a user of Ring's wrap-json-body middleware, I've had an issue where I send up invalid JSON and receive a 500 status code since an exception is thrown server-side. This is less than ideal since it seems to indicate that the problem is server-side rather than client side. As a quick, hopefully temporary fix, I've written some middleware to wrap it with some exception handling to return a 400 status code with helpful response body.

Here's the code:

{% highlight clojure %}
(ns clj-todo-list.middleware
  (:require [ring.middleware.json :as ring-json])
  (:import [com.fasterxml.jackson.core JsonParseException]))
(defn wrap-json-body
  [handler]
  (fn [request]
    (try
      ((ring-json/wrap-json-body handler) request)
    (catch JsonParseException e
      {:status 400
       :body "Could not deserialize JSON"}))))
{% endhighlight %}

The major catch is that this relies on an implementation detail of a library used by another library used by another library (i.e., Jackson, Cheshire, and Ring Middleware) making this approach fairly fragile.
