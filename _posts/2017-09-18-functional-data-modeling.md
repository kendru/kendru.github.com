---
layout: post
title: "Functional Data Modeling"
description: "Tutorial on using ClojureScript's basic features to manipulate data and create a DSL using only functions for constructing and enriching data."
category: ClojureScript
tags: ["programming", "clojurescript", "modeling"]
---

ClojureScript is a language designed for working with data. In this post, we'll
look at how to use the builtin collection library as well as a few design patterns
to model a domain. ClojureScript places a strong emphasis on relying on
generic collection types and the standard functions that operate on them rather
than creating highly specialized functions that only work on a single type of
object. The object-oriented approach, which most mainstream languages encourage,
is to create objects that encapsulate both the data and behaviour of a specific
type of "thing". The practice that ClojureScript encourages, however, is to
separate functions and data. Data is pure information, and functions are pure
transformations of data.

![Functions and Data](/img/functions-and-data.png)

_Functions and Data_

## Modeling a Domain

Say that are creating an analytics app. Before we get started, we want to model
the type of objects that we will be working with. If we were using a statically
typed language, we would probably start by writing type definitions. Even if
we were working in JavaScript, we would likely define "classes" for the objects
that we will be working with. As we define these objects, we would have to think
about both the data that they contain and the operations that they support. For
example, if we have a `User` and a `ClickEvent`, we might need the operation,
`User.prototype.clickEvent()`.

![Users and Actions](/img/analytics-domain.png)

_Our analytics domain deals users and their actions_

With ClojureScript, we will consider data and functions separately. This
approach ends up being flexible, as we will see that most of the operations that
we want to perform on the data are simple and re-usable. In fact, it is common
to find that the exact operation that you need is already part of the standard
library. Ultimately, the combination of the concision of code and the richness
of the standard library means that we write fewer lines of code than we would in
JavaScript, which leads to more robust and maintainable applications.

## Domain Modeling with Maps and Vectors

We are now quite familiar with what maps and vectors as well as some of the
collection and sequence operations that can be used on them. Now we can put them
in practice in a real domain: an analytics dashboard. The main concepts that we
need to model are _user_, _session_, _pageview_, and _event_, and the
relationships between these models are as follows:

- A user has one or more sessions
- A session has one or more pageviews and may belong to a user or be anonymous
- A pageview has zero or more events

We now know enough to create some sample data. Let's start at the "bottom" with
the simplest models and work our way up to the higher-level models. Since an
_event_ does not depend on any other model, it is a good place to start.

<script async src="//pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
<ins class="adsbygoogle"
     style="display:block; text-align:center;"
     data-ad-layout="in-article"
     data-ad-format="fluid"
     data-ad-client="ca-pub-6265787006533161"
     data-ad-slot="3706397953"></ins>
<script>
(adsbygoogle = window.adsbygoogle || []).push({});
</script>

### Modeling Events

An event is some action that the user performs while interacting with a web
page. It could be a _click_, _scroll_, _field entry_, etc. Different events may
have different properties associated with them, but they all have at least a
type and a timestamp.

#### Modeling an Event

{% highlight clojure %}
(def my-event {:type :click               ;;  <1>
               :timestamp 1464362801602
               :location [1015 433]       ;;  <2>
               :target "#some-elem"})
{% endhighlight %}

1. Every event will have `:type` and `:timestamp` entries
2. The remaining entries will be specific to the event type

When we think of data types like _event_ in ClojureScript, we usually create at
least a mental schema of the data type. To enforce a schema, we could use
[clojure.spec](https://clojure.org/about/spec) or
[plumatic/schema](https://github.com/plumatic/schema), but for now we will just
enforce the "shape" of our data structures by convention. That is, we will ensure
that whenever we create an event, we create it with a timestamp and a type. In fact,
it is a common practice to define on or more functions for constructing the new
data types that we create. Here is an example for how we might do this with
_events_:

#### Using a Constructor Function

{% highlight clojure %}
cljs.user=> (defn event [type]
              {:type type
               :timestamp (.now js/Date)})
#'cljs.user/event

cljs.user=> (event :click)
{:type :click, :timestamp 1464610050488}
{% endhighlight %}

This function simply abstracts the process of creating a new object that follows
the convention that we have established for events. We should also create a
constructor function for click events specifically:

{% highlight clojure %}
cljs.user=> (defn click [location target]
              (merge (event :click)
                     {:location location, :target target}))
#'cljs.user/click

cljs.user=> (click [644 831] "#somewhere")
{:type :click,
 :timestamp 1464610282324,
 :location [644 831],
 :target "#somewhere"}
{% endhighlight %}

The only thing about this code that might be unfamiliar is the use of the
`merge` function. It takes at least two maps and returns a new map that is the
result of adding all properties from each subsequent map to the first one. You
can think of it as `conj`-ing every entry from the second map onto the first.

### Modeling Pageviews

With events done, we can now model pageviews. We will go ahead and define
a constructor for pageviews:

#### Modeling a Pageview

{% highlight clojure %}
cljs.user=> (defn pageview
              ([url] (pageview url (.now js/Date) [])) ;; <1>
              ([url loaded] (pageview url loaded []))
              ([url loaded events]
                {:url url
                 :loaded loaded
                 :events events}))

cljs.user=> (pageview "some.example.com/url")          ;; <2>
{:url "some.example.com/url",
 :loaded 1464612010514,
 :events []}

cljs.user=> (pageview "http://www.example.com"         ;; <3>
                      1464611888074
                      [(click [100 200] ".logo")])
{:url "http://www.example.com",
 :loaded 1464611888074,
 :events [{:type :click,
           :timestamp 1464611951519,
           :location [100 200],
           :target ".logo"}]}
{% endhighlight %}

1. Define `pageview` with 3 arities
2. `pageview` can be called with just a URL
3. ...or with a URL, loaded timestamp, and vector of events

Just like we did with events, we created a constructor to manage the details of
assembling a map that fits our definition of what a _Pageview_ is. One different
aspect of this code is that we are using a multiple-arity function as the
constructor and providing default values for the `loaded` and `events` values
when they are not supplied. This is a common pattern in ClojureScript for
dealing with default values for arguments.

### Modeling Sessions

Moving up the hierarchy of our data model, we now come to the
_Session_. Remember that a session represents one or more consecutive pageviews
from the same user. If a user leaves the site and comes back later, we would
create a new session. So the session needs to have a collection of pageviews as
well as identifying information about the user's browser, location, etc.

{% highlight clojure %}
cljs.user=> (defn session
              ([start is-active? ip user-agent] (session start is-active? ip user-agent []))
              ([start is-active? ip user-agent pageviews]
                {:start start
                 :is-active? is-active?
                 :ip ip
                 :user-agent user-agent
                 :pageviews pageviews}))

cljs.user=> (session 1464613203797 true "192.168.10.4" "Some UA")
{:start 1464613203797, :is-active? true, :ip "192.168.10.4", :user-agent "Some UA", :pageviews []}
{% endhighlight %}

There is nothing new here. We are simply enriching our domain with more types
that we will be able to use in an analytics application. The only piece that
remains is the _User_, which I will leave as an exercise for you.

We now have a fairly complete domain defined for our analytics
application. Next, we'll explore how we can interact with it using primarily
functions from ClojureScript's standard libary. Below is a sample of what some
complete data from our domain looks like at this point. It will be helpful to
reference this data as we move on.

#### Sample data for an analytics domain

{% highlight clojure %}
;; User
{:id 123
 :name "John Anon"
 :sessions [

   ;; Session
   {:start 1464379781618
    :is-active? true
    :ip 127.0.0.1
    :user-agent "some-user-agent"
    :pageviews [

      ;; Pageview
      {:url "some-url"
       :loaded 1464379918936
       :events [

         ;; Event
         {:type :scroll
          :location [403 812]
          :distance 312
          :timestamp 1464380102036}

         ;; Event
         {:type :click
          :location [644 112]
          :target "a.link.about"
          :timestamp 1464380117760}]}]}]}
{% endhighlight %}
