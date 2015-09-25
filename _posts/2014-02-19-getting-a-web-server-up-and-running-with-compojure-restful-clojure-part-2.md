---
layout: post
title: "Getting a web server up and running with Compojure | RESTful Clojure, Part 2"
description: "With our configuration and build automated, now we set up a basic
- but functional - web server"
category: restful-clojure
tags: ["clojure", "REST", "Compojure", "tutorial"]
---

This tutorial is the second in a series on building a RESTful web service in
Clojure. In the
[first tutorial](/restful-clojure/2014/02/16/writing-a-restful-web-service-in-clojure-part-1-setup/),
we set up a development environment using Puppet and Vagrant, and we created
a skeleton Leiningen project. With all of the prerequisites taken care of, now
it's time to get our hands dirty with some code!

## Ring
If you have done any web programming in Clojure before, you have probably used
[Ring](https://github.com/ring-clojure/ring). Ring provides a very thin
abstraction over the HTTP request->response cycle. It is similar Ruby's Rack or
the Java servlets specification, allowing you to write applications that
respond to HTTP requests without having to deal with the low-level details of
sockets, request parsing, etc.

In order to make use of Ring in our application, we need to add it as
a dependency in Leiningen's `project.clj`.

{% highlight clojure %}
; project.clj
(defproject restful-clojure "0.1.0-SNAPSHOT"
  ; ...project settings...

  ; The :dependencies key maps to a vector containing all dependencies
  ; necessary for our project. The dependency on Clojure itself should
  ; have already been added by Leiningen. We will add Ring, the Jetty
  ; adapter (so we can start up a web server to serve our application),
  ; and Compojure, which we will use later on in this tutorial.
  :dependencies [[org.clojure/clojure "1.5.1"]
                 [ring/ring-core "1.2.1"]
                 [ring/ring-jetty-adapter "1.2.1"]
                 [compojure "1.1.6"]])
{% endhighlight %}

With our dependencies declared, let's fire up a REPL and build a quick server
(Leiningen will download the dependencies before launching the repl):

{% highlight bash %}
lein repl
{% endhighlight %}
{% highlight clojure %}
(use 'ring.adapter.jetty)
; => nil
(defn app-handler [request]
  {:status 200
   :headers {"Content-Type" "text/html"}
   :body "Hello from Ring"})
; => #'user/app-handler
(run-jetty app-handler {:port 3000})
{% endhighlight %}

That's it! We now have a Ring application up and running, albeit a pretty
boring one. If you visit `localhost:3000` in a web browser, you should see the
message, "Hello from Ring" displayed.

### Ring basics

A Ring application consists of a _handler_ function, which is just a normal
Clojure function that takes a request map and returns a response map. In the
case of the `app-handler` function above, we're binding the request map to the
`request` var and returning a response map with `:status`, `:headers`, and
`:body` keys. Let's go back to the REPL and create a new server that responds
by printing out the request map received as a Clojure map:

{% highlight clojure %}
(use 'ring.adapter.jetty)
; => nil
(defn app-handler [request]
  {:status 200
   :headers {"Content-Type" "text/plain;=us-ascii"}
   :body (str request)})
; => #'user/app-handler
(run-jetty app-handler {:port 3000})
{% endhighlight %}

Now when you visit `localhost:3000` with a browser (or any other HTTP client),
you should see your HTTP request printed as a Clojure map.

As I said above, Ring is a thin abstraction over HTTP. Consider the following
HTTP request and response below (with some formatting applied to the response
body):

    GET /page HTTP/1.1
    Host: localhost
    Accept: text/plain
    
    HTTP/1.1 200 OK
    Date: Sat, 22 Feb 2014 05:21:47 GMT
    Content-Type: text/plain;charset=ISO-8859-1
    Content-Length: 350
    Server: Jetty(7.6.8.v20121106)
    
    {:ssl-client-cert nil,
    :remote-addr "0:0:0:0:0:0:0:1",
    :scheme :http,
    :request-method :get,
    :query-string nil,
    :content-type nil,
    :uri "/page",
    :server-name "localhost",
    :headers {"accept" "text/plain", "host" "localhost"},
    :content-length nil,
    :server-port 80,
    :character-encoding nil,
    :body #<HttpInput org.eclipse.jetty.server.HttpInput@2523a57f>}
     

 

<!-- TODO: Add graphic comparing HTTP request to request map -->

You can see how Ring converted our request into a map, which is just a Clojure
datatype. We now have all of the power of Clojure to pull apart that request to
generate a response.

Okay, so we can create a "Hello World" web server in Clojure. Big deal. Let's
move on to something more useful.

## Meet Compojure

Ring is a great library, but it is pretty low-level, and building a web app
directly on top of Ring would be only mildly less tedious than herding cats
(and probably not nearly as entertaining). One of the beautiful aspects of
functional programming in general, and Clojure in particular, is that you can
work with very high-level abstractions that are very close to the business
domain of whatever application you are writing. Why bother when we can work
more abstractly?

Enter [Compojure](https://github.com/weavejester/compojure), an excellent
library from [James Reeves](http://www.booleanknot.com/) that abstracts away
some of the details of Ring and provides us with a simple interface for routing
requests. If you have ever used [Sinatra](http://www.sinatrarb.com/) in Ruby,
[Flask](http://flask.pocoo.org/) in Python, or
[Slim](http://www.slimframework.com/) in PHP, then Compojure will seem pretty
familiar.

Before we dive in, there are a couple more options that we should add to
`project.clj` to make our development a little smoother with Ring and
Compojure.

{% highlight clojure %}
; project.clj
(defproject restful-clojure "0.1.0-SNAPSHOT"
  ; ...project settings...

  ; The lein-ring plugin allows us to easily start a development web server
  ; with "lein ring server". It also allows us to package up our application
  ; as a standalone .jar or as a .war for deployment to a servlet contianer
  ; (I know... SO 2005).
  :plugins [[lein-ring "0.8.10"]]

  ; See https://github.com/weavejester/lein-ring#web-server-options for the
  ; various options available for the lein-ring plugin
  :ring {:handler restful-clojure.handler/app
         :nrepl {:start? true
                 :port 9998}}
  :profiles
  {:dev {:dependencies [[javax.servlet/servlet-api "2.5"]
                        [ring-mock "0.1.5"]]}})
{% endhighlight %}

Instead of a lengthy introduction, let's start by playing with some code!
Create `restful-clojure/src/restful_clojure/handler.clj`, and use the code from
below to create an application that can count either up or down.

{% highlight clojure %}
(ns restful-clojure.handler
  (:use compojure.core)
  (:require [compojure.handler :as handler]))

(defn- str-to [num]
  (apply str (interpose ", " (range 1 (inc num)))))

(defn- str-from [num]
  (apply str (interpose ", " (reverse (range 1 (inc num))))))

(defroutes app
  (GET "/count-up/:to" [to] (str-to (Integer. to)))
  (GET "/count-down/:from" [from] (str-from (Integer. from))))

{% endhighlight %}

## Starting a web server

Now that we have a counting API, let's get the web server started. Instead of
running locally, we'll start the server on our VM to mimick a more
production-like environment. Change into the root of the repository (the
directory that contains the `Vagrantfile`, and log onto your VM:

{% highlight bash %}
vagrant up
vagrant ssh
cd /vagrant
lein ring server-headless
{% endhighlight %}

If all goes well, you should be able to visit
`http://192.168.33.10:3000/count-up/10` on your host machine and see that your
web server does indeed know how to count. One thing that you may have noticed
when you started the server on your VM was the line,

```
Started nREPL server on port 9998
```

[nREPL](https://github.com/clojure/tools.nrepl) allows us to connect to a REPL
remotely and interact with our server as it is running. If you want to play
around with it, you can connect to the REPL on your VM using a number of tools,
including Leiningen:

{% highlight bash %}
lein repl :connect 192.168.33.10:9998
{% endhighlight %}

{% highlight clojure %}
(in-ns 'restful-clojure.handler)
(defn str-to [num] "I forgot how to count!")
{% endhighlight %}

Now if you reload `http://192.168.33.10:3000/count-up/10` in your browser, you
should see "I forgot how to count!" Pretty cool, huh? Just remember that in the
immortal words of Uncle Ben, "With great power comes great responsibility."

## Test-driven development

I am a firm believer in the value of writing tests first. There are several
huge advantages that I can identify in TDD:
- When you write the test first, it forces you to think through what the
expected functionality is
- Tests serve as technical documentation that describe by example what any
given portion of code is supposed to do
- It's satisfying to see _your_ code transform a failing test into a passing
test.

That said, tests are not an excuse to not think about the code you're writing.
Just because a test passes does not mean that the code under test is elegant or
efficient. Tests are just a tool in our toolbox to help us craft solid,
reliable applications.

Instead of writing tests for our "dummy" counting application, let's go ahead
and write several tests for the real application that we'll begin building in
the next tutorial. We know that we are going to have a "users" resource as well
as a "lists" resource that should both return JSON, so let's go ahead and write
the tests to check that loading "/users" and "/lists" return an HTTP 200
response code with a JSON response body. Remove the existing test in
`tests/restful_clojure/` and create `tests/restful_clojure/handler_test.clj`
with the code below.

{% highlight clojure %}
(ns restful-clojure.handler-test
  (:use clojure.test
        ring.mock.request  
        restful-clojure.handler))

(deftest test-app
  (testing "users endpoint"
    (let [response (app (request :get "/users"))]
      (is (= (:status response) 200))
      (is (= (get-in response [:headers "Content-Type"]) "application-json"))))

  (testing "lists endpoint"
    (let [response (app (request :get "/lists"))]
      (is (= (:status response) 200))
      (is (= (get-in response [:headers "Content-Type"]) "application-json"))))

  (testing "not-found route"
    (let [response (app (request :get "/bogus-route"))]
      (is (= (:status response) 404)))))
{% endhighlight %}

We're using the
[clojure.test](http://richhickey.github.io/clojure/clojure.test-api.html)
framework for testing because it probably has the lowest barrier to entry of
any of the Clojure testing frameworks out there. If you're used to a tool like
Rspec and would like something a little more expressive, I'd recommend looking
into [Midje](https://github.com/marick/Midje). Now let's run those tests and
watch them fail.

{% highlight bash %}
lein test
{% endhighlight %}

```
lein test restful-clojure.handler-test

lein test :only restful-clojure.handler-test/test-app

FAIL in (test-app) (handler_test.clj:9)
users endpoint
expected: (= (:status response) 200)
  actual: (not (= nil 200))

... MORE OUTPUT ...

Ran 1 tests containing 5 assertions.
5 failures, 0 errors.
```

Perfect - we now know exactly what we need to implement in order to get these
tests to pass. In the next tutorial, we'll start implementing a few features of
our RESTful web service, including authentication and authorization, a data
model, and a full RESTful interface for our _users_ and _lists_ resources (all
with tests, of course).

---

I would love to get your feedback on these tutorials! Useful or not useful?

### Go To

- [Part 1: Setup](/restful-clojure/2014/02/16/writing-a-restful-web-service-in-clojure-part-1-setup/)
- Part 2: Getting a web server up and running
- [Part 3: Building out the web service](/restful-clojure/2014/03/01/building-out-the-web-service-restful-clojure-part-3/)
- [Part 4: Securing the service](/restful-clojure/2015/03/13/securing-service-restful-clojure-part-4/)
