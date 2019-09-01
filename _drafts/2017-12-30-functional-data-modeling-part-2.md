---
layout: post
title: "Functional Data Modeling, Part 2"
description: "Tutorial on using ClojureScript's basic features to manipulate data and create a DSL using only functions for constructing and enriching data."
category: ClojureScript
tags: ["programming", "clojurescript", "modeling"]
---

In the last post, we looked at modeling a simple analytics domain using
ClojureScript. We dealt primarily with what the functional analogs of what are
known in OO literature as creational design patterns - things like factories and
builders. These types of patterns are perhaps even more important in functional
programming, since a key attribute of functional programming is designing
versatile immutable data structures.

However, creating data is only half the job of modeling a domain. We have
essentially modeled the nouns in our system but not the verbs. With the
exception of professional golf, there is not a single real-world domain in which
nothing happens, so let's complete our domain modeling tutorial by adding some
behaviour to our analytics model. After this, we'll have the tools that we need
to eloquently express an entire domain using functional programming techniques.

## Replacing Design Patterns with FP

Most object-oriented programmers are familiar with the classic
[Gang of Four](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612)
book on design patterns - if they don't actually have a copy on their shelf.
This book was something of a breakthrough in that it elucidated some of the
common patterns that programmers used in nontrivial object-oriented codebases.
It was both loved and reviled: loved by those who saw it as a way to build a
common vocabulary to talk about high-level software design and hated by those
who argued that the mere need of a design pattern is indicative that a
programming language is deficient. We don't need better patterns, they argue,
but better languages.

While the need to reach for a design pattern for everything is probably an
indication that we picked the wrong tool for the job, there is use in keeping
a common vocabulary of proven ways to handle certain types of problems. We
may have simply traded the term "design pattern" for "best practice". Whatever
term we use, there are certain practices that we follow whether consciously
or not that inform how we write code. When coming to functional programming,
we find that there are fewer, more general patterns to use.

In the last post, we essentially replaced the factory pattern with objects that
return simple maps, like so:

{% highlight clojure %}
(defn event [type]
  {:type type
   :timestamp (.now js/Date)})
{% endhighlight %}

We saw how to take advantage of variable-arity functions (ones that can take
a variable number of arguments) to set default values for common parameters and
use `merge` to compose maps together. Now we'll look at some more ways to build
up data over time, similar to the Builder pattern. Then, we will look at a
couple of common idioms for "modifying" data (actually creating modified
copies).

## Building Data

In ClojureScript, it is common to begin with very small data types in a domain
then developing a language that expresses the core of the domain. From there,
most of the interesting things that we will build are simply compositions of
these functions and data. As part of the core of our domain, we want to define
some functions that will allow us to more easily construct the objects from the
last tutorial. For example, we can introduce a couple of `with-*` functions that
add one or two entries to a map then rewrite some of the more complex functions
in terms of these.

For example:

{% highlight clojure %}
cljs.user=> (defn with-location [event loc]
              (assoc event :location loc))
#'cljs.user/with-location

cljs.user=> (with-location (event :click) [312 64])
{:type :click,
 :timestamp 1514702429058,
 :location [312 64]}

cljs.user=> (defn with-target [event selector]
              (assoc event :target selector))
#'cljs.user/with-target

cljs.user=> (with-target (event :hover) "#ad-102")
{:type :hover,
 :timestamp 1514702456864,
 :target "#ad-102"}
{% endhighlight %}

As we can see, the `assoc` function simply sets a key within a map to a certain
value. As with all operations on immutable data structures, we get a new copy,
and the original is left unchanged. With these functions, we can now rewrite the
"click" function:

{% highlight clojure %}
(defn click [location target]
  (-> (event :click)
      (with-location location)
      (with-target target)))
{% endhighlight %}

Here we used the functions we just wrote, along with the `->` (read as
"thread-first") macro, to refactor the "click" function to be a bit more
readable. The `->` macro takes an expression and evaluates it then passes the
result as the first argument in the next expression, which it then evaluates and
passes on to the next expression, etc. The code above us functionally equivalent
to the following:

{% highlight clojure %}
(defn click [location target]
  (with-target           ;; Eval'd third
    (with-location       ;; Eval'd second
      (event :click)     ;; Eval'd first
      location)
    target))
{% endhighlight %}

`->` allows us to write nested function calls in a much cleaner, more natural
manner that also helps the code read a bit like natural language: "Make an
event with this location and with this target".

### Bestowing Semantics

Sometimes it is useful to take an existing data structure and abstract it with
a couple of functions. For instance, we are assuming that `location` is a
2-element vector with x and y coordinates, but we could just as easily have
represented it as a map, such as `{:x 42, :y 509}` or even a vector whose
first element is a polar angle and second element is a magnitude. Abstracting
this data type with a constructor and a couple of accessors gives us the freedom
to change the underlying representation, and it helps to document our code.
Let's take a stab at promoting "location" to a first-class concept.

{% highlight clojure %}
cljs.user=> (defn location [x y] [x y])  ;; Pack the arguments in to a 2-element vector
#'cljs.user/location
cljs.user=> (def x-val first)            ;; x-val is just a synonym for "first"
#'cljs.user/x-val
cljs.user=> (def y-val second)           ;; ...and y-val for "second"
#'cljs.user/y-val

cljs.user=> (def my-click (click (location 234 585) "#foo")) ;; Client code should now use the location funtion
#'cljs.user/my-click

cljs.user=> (y-val (:location my-click)) ;; Convenience function to extract data
585
{% endhighlight %}

If we later wanted to use a map, all that we would have to change would be these
three functions:

{% highlight clojure %}
(defn location [x y] {:x x, :y y})
(def x-val :x)
(def y-val :y)
{% endhighlight %}

With just a few functions we have imported a meaning onto a general data type.
We have gained clarity and made our code more self-documenting without
sacrificing serializability of our data model or introducing too much
complexity.

## Transforming Data

The simple patterns that we have seen so far work quite well, but they will not
cut it for all but the simplest cases. We will often need to "reach into" nested
data or perform updates based on the current value of an object. For these, we
need a couple more tools. In the case of nested data, there are a couple of
functions in the standard library that we will look at: `get-in` and `assoc-in`.
This pair of functions lets us operate on nested properties of an object.

For instance, if we wanted to get the landing URL of a particular session, we
would need to find the URL of the first pageview of that session - the perfect
job for `get-in`:

{% highlight clojure %}
(defn landing-url [session]
  (get-in
    session          ;; Collection to traverse
    [:pageviews 0])) ;; Path to look up
{% endhighlight %}

`get-in` works by drilling down into a collection based on the path that we give
it. In this case, it looks for the value at the `:pageview` key, then for the
0th element in this collection. Finally, it evaluates to the value at that path
or `nil` if it could not successfully walk the entire path.

Similarly, to set a nested value, we use the related function, `assoc-in`:

{% highlight clojure %}
(defn redact-landing-url [session]
  (assoc-in
    session          ;; Collection to traverse
    [:pageviews 0]   ;; Path at which to set new value
    "REDACTED"))     ;; Value to set
{% endhighlight %}

As expected, this will replace the landing URL with "redacted". If `assoc-in`
cannot find the next element in its path, it creates a new map whose only key
is the next segment in the path, which may not be exactly what we want:

{% highlight clojure %}
cljs.user=> (redact-landing-url {})
{:pageviews {0 "REDACTED"}}
{% endhighlight %}

Finally, let's look at 
