---
title: ! 'Beautiful Haskell: Transposing a List'
---
Recently I've been reading Real World Haskell to improve my functional programming skills and learn Haskell. One thing that really appeals to me about a programming language is how well it encourages elegant solutions for a given problem and so far Haskell has been excellent at encouraging elegant solutions for the exercises I've attempted.

One of the exercises in the fourth chapter of Real World Haskell is to write a function that will transpose a string split on newlines. For example, given the input "Hello\nWorld", the function should return "HW\neo\nrl\nll\nod". This isn't a particularly hard problem to solve in an imperative language but the functional solution I came up with seems much more elegant than any imperative solution.

Here it is:

{% highlight haskell %}
transpose :: String -> String
transpose input = unlines (inner (lines input))
    where
          inner :: [String] -> [String]
          inner ls | not (any null ls) = map head ls : inner (map tail ls)
          inner _                      = []
{% endhighlight %}

Transpose's expression just splits the input string by newline, runs it through the inner function, and then joins the resulting string list together with newlines. Inner's first expression runs only if none of the lines passed are empty. It creates a list of all of the first characters in the lines passed in and then prepends those to the results of running the function recursively on the rest of each of the lines. When at least one of the lines passed in are empty, inner's second expression is called which just returns back an empty list.

For me, this is a beautiful solution because the code is extremely minimal yet understandable (at least if you know Haskell). A similar solution in many imperative languages would be burdened with boilerplate and would detail the mechanics of transposition rather than simply describing it.
