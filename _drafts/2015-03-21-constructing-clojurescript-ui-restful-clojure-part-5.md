---
layout: post
title: "Constructing a ClojureScript UI | RESTful Clojure, Part 5"
description: "Create a simple front-end for the Clojure shopping list application."
category: restful-clojure
tags: ["clojurescript", "reagent", "ui", "tutorial"]
---

At this point we have a fully-functional Clojure REST API that will allow us to
build a simple shopping list application. And - no surprises here - we will be
building the UI as a ClojureScript application. While I have been using Clojure
for server-side applications for a few years now, I have only recently begun
exploring ClojureScript. While the language still feels a little rough around
the edges, the innovation that is happening in the ClojureScript community is
incredible. If you have not already watched David Nolen's excellent talk,
[The Functional Final Frontier](http://www.infoq.com/presentations/om-clojurescript-facebook-react),
please do that now. In fact, do that instead of reading the rest of this blog -
it's that good.

With a plug like that, you might guess that we would use [om](https://github.com/omcljs/om)
to build our front end. Good guess. But wrong. We'll instead be using [Reagent](http://holmsand.github.io/reagent/),
another popular ClojureScript wrapper around [React](http://facebook.github.io/react/).
Both projects are very well-written and easy to use, but for the purpose of this
tutorial, I decided to go with Reagent for a couple of reasons:

#### A simpler data model
Om components receive their data via [cursors](https://github.com/omcljs/om/wiki/Cursors),
which are an interesting idea, but come with their own complexities. Moreover,
for communicating between components, the best practices are to use `core.async`
channels or an observable kind of cursor called a _reference cursor_. While these
tools are useful, they also come with their own additional issues that are too
much to cover in a simple tutorial.

#### A more obvious API
Reagent hides some of the implementation details that are more visible in om. In
om, creating a useful component usually involves returning a reified instance of
one or more lifecycle protocols, including either `IRender` or `IRenderState`. In
Reagent, you return a data structure that looks like, well, lisp-ified HTML. There
are some more nit-picky things that have tripped me up a couple of times with
om, such as having to convert a ClojureScript map to a JavaScript object before
passing them as attributes to a component. To illustrate, here are two comparable
components in om and Reagent.

{% highlight clojure %}
(defn om-list [items owner]
  (reify
  om/IRender
  (render [_]
    (apply dom/ul #js {:className "my-list"}
      (for [item items]
        (dom/li nil item))))))

(defn reagent-list [items]
  [:ul.my-list
    (for [item items]
      [:li item])])
{% endhighlight %}

To be fair, these minor annoyances that I have with om are easy enough to build
an abstraction over, and for a large application, I would most likely choose om
_because of_ its more transparent wrapping of React.

## Getting down to business

Now that you have heard me wax philosophical about a couple of ClojureScript UI
frameworks, let's actually write some code... almost. First, a quick sketch of
what we want the final app to look like:

![List app mockup](/img/reagent-app-mockup1.png)
_Basic view_

If you remember from the [last tutorial](/restful-clojure/2015/03/13/securing-service-restful-clojure-part-4/),
admin users can add products to the application, so we probably want to provide
them with an interface for adding products:

![List app admin view](/img/reagent-app-mockup2.png)
_Admin view_

You may be thinking, "It's a good thing that this guy is not a designer", and I
would be inclined to agree wholeheartedly. Thankfully, this is not a design
tutorial but more of a test of the API that we built, so let's move on.

### Project Scaffolding

Now that we know what we want to create, let's set up the project. In the root
directory of the repo, let's create a new Leiningen project using the
[Figwheel](https://github.com/bhauman/lein-figwheel) project, which will give us
automatic code compilation and reloading as well as in-browser compiler errors.
There is a leinengen template for a skeleton figwheel project, which supports
including some minimal reagent scaffolding as well. Double win.

{% highlight bash %}
lein new figwheel restful-cljs -- --reagent
{% endhighlight %}

As part of the lein-figwheel template, a primary source directory was created at
`src`, and a dev specific directory was created at `dev_src`. This is so that we
can keep figwheel-specific reloading code (and other code that should not make
it to production) out of our main codebase. Let's take a look at the default
namespace that was created at `dev_src/restful_cljs/dev.cljs`:

{% highlight clojure %}
(ns restful-cljs.dev
    (:require
     [restful-cljs.core]
     [figwheel.client :as fw]))

(fw/start {
  :websocket-url "ws://localhost:3449/figwheel-ws"
  :on-jsload (fn []
               ;; (stop-and-start-my app)
               )})
{% endhighlight %}

And let's take a peek at the production code  skeleton that was generated:

{% highlight clojure %}
(ns ^:figwheel-always restful-cljs.core
    (:require
              [reagent.core :as reagent :refer [atom]]))

(enable-console-print!)

(println "Edits to this text should show up in your developer console.")

;; define your app data so that it doesn't get over-written on reload

(defonce app-state (atom {:text "Hello world!"}))

(defn hello-world []
  [:h1 (:text @app-state)])

(reagent/render-component [hello-world]
                          (. js/document (getElementById "app")))
{% endhighlight %}

The important piece of code to note here is the `(defonce app-state ...)` The
`defonce` protects the app state from getting re-defined when figwheel automatically
reloads the code.

Let's enter the project directory, start up figwheel, and begin coding:

{% highlight bash %}
cd restful-cljs
lein figwheel
{% endhighlight %}

### Building the UI

Reagent, and it's React underpinnings, support a one-way data flow, so our app's
data flow will look like this:

Server -> Application State -> UI Components

When the user initiates an event, such as clicking the "Add Product" button, we
will fire an action that updates the application state and performs the necessary
action on the server. For a larger app, you would definitely want to implement
some error handling and alert the user if some action on the server failed.

We'll break the app out into the following namespaces:

- `core` - Bootstrap the app
- `api` - Server API interaction
- `state` - Define app state and the actions that may be performed on it
- `ui` - Main UI components
- `ui-admin` - UI components for admin interface
- `util` - Miscellaneous helper functions

Let's start by filling out the `state` namespace with a placeholder for our app
state and a few basic actions.

{% highlight clojure %}
(ns restful-cljs.state
  (:require-macros [reagent.ratom :refer [reaction]])
  (:require [reagent.core :as reagent :refer [atom]]
            [restful-cljs.util :refer [map-values]]))

(defonce app (atom {:status {:loading? false
                             :logged-in? false}
                    :user {:name ""
                           :token ""
                           :level "user"}
                    ;; Index products by id for efficient retrieval
                    :products {1 {:id 1 :title "Ramen" :description "Cheap soup"}
                               2 {:id 2 :title "Caviar" :description "Good for snacking"}
                               ;; ...
                               }}
                    ;; Index lists by id too
                    :lists {1 {:id 1
                              :title "Friday Shopping"
                               ;; Store denormalized products
                               :products [{:id 1 :title "Ramen" :description "Cheap soup"}
                                          {:id 2 :title "Caviar" :description "Good for snacking"}]}
                            ;; ...
                            }})

;; Reagent "reactions" will update themselves whenever
;; a Reagent atom or another reaction that they contain
;; is updated.
(defonce status
  (reaction (:status @app)))

(defonce lists
  (reaction (:lists @app)))

(defonce products
  (reaction (:products @app)))

(defonce active-list
  (reaction (some #(when (:active? %) %) @lists)))

(defonce products-not-in-active-list
  (reaction
    (let [product-ids-in-list (into #{} (map :id (:products @active-list)))]
      (filter #(not (contains? product-ids-in-list (:id %))) @products))))

(defn get-next-id-for [k]
  (inc (reduce max (map :id (get @app k)))))

(defn remove-product-from-list [lst id]
  (update-in lst [:products]
    (fn [products]
      (remove #(= id (:id %)) products))))

(defmulti dispatch! (fn [tag _] tag))

(defmethod dispatch! :create-list [_ opts]
  (let [{:keys [title]} opts
        id (get-next-id-for :lists)]
    (swap! app update-in [:lists] conj {:id id
                                        :title title
                                        :products []})))

(defmethod dispatch! :set-active-list [_ opts]
  (let [{:keys [list-id]} opts]
    (swap! app update-in [:lists] (fn [lists]
                                    (map #(assoc % :active? (= list-id (:id %))) lists)))))

(defmethod dispatch! :add-product-to-list [_ opts]
  (let [{:keys [list-id product]} opts]
    (swap! app update-in [:lists] (fn [lists]
                                    (map #(if (= list-id (:id %))
                                            (update-in % [:products] conj product) %) lists)))))

(defmethod dispatch! :create-product [_ opts]
  (let [{:keys [title description]} opts
        id (get-next-id-for :products)]
    (swap! app update-in [:products] conj {:id id
                                           :title title
                                           :description description})))

(defmethod dispatch! :delete-product [_ opts]
  (let [{:keys [product-id]} opts]
    ; Remove product from any list that contains it
    (swap! app update-in [:lists] #(map remove-product-from-list %))
    ;; Remove from the products store as well
    (swap! app update-in [:products] (fn [products]
                                        (remove #(= product-id (:id %)) products)))))

(defmethod dispatch! :login-user [_ opts]
  (let [{:keys [email password]} opts]
    (swap! app assoc :logged-in true)
    (swap! app update-in [:user] assoc :name "Test User")))
{% endhighlight %}

We will need to change these actions once we get the server integration in place,
but this should be enough to play around with as we create the UI. Instead of
letting components update the app state directly, we introduce a `dispatch!`
multimethod that will dispatch to a function that acts on the app state. For all
but the simplest of apps, it is good practice to have well-defined *domain operations*
that decouple UI components from your business logic.

One of the cool features of Reagent that we are using is a `reaction`. A Reagent reaction
is a computation that may dereference a Reagent atom or another reaction and will automatically
be updated when the underlying value changes. This enables us to hook up a FRP graph with our
application (think spreadsheet). See [re-frame](https://github.com/Day8/re-frame) for an example
of how to use an FRP-like flow to structure Reagent applications.

We are using a denormalized application state, where products will be duplicated
between the products map and the products in individual lists. This will place
slightly more burden on our state-updating code but will simplify the UI
components. Speaking of the UI, let's look at the `ui` namespace. We'll start
with a top-level `page` component that will contain the list selector, new
list form, product list, and product add widget. We'll also take out the placeholder
code from the `core` namespace and load our `page` component into the document
instead.

{% highlight clojure %}
;; src/restful_cljs/ui.cljs
(ns restful-cljs.ui
  (:require [reagent.core :as reagent :refer [atom]]
            [restful-cljs.state :as state :refer [dispatch!]]))

(defn list-form []
  [:p "TODO: New List"])

(defn lists-selector []
  [:div {:class "lists"}
    [:h3 "Lists"]
    [list-form]])

(defn product-list []
  [:p "TODO: Add product"])

(defn page []
  [:div {:class "app"}
    [:h1 "My Lists"]
    [lists-selector]
    [product-list]])

;; src/restful_cljs/core.cljs
(ns ^:figwheel-always restful-cljs.core
    (:require [reagent.core :as reagent :refer [atom]]
              [restful-cljs.ui :refer [page]]))

(enable-console-print!)

(reagent/render-component [page]
                          (. js/document (getElementById "app")))
{% endhighlight %}

The completed code for the UI is is the restful clojure GitHub repo, and it is
pretty straightforward, so we will not go into it in great detail here. However,
let's go over one of the completed components:

{% highlight clojure %}
(defn product-selector []
  ;; Init local state
  (let [selector-active? (atom false)]
    ;; And return a component-building function
    (fn []
      [:div
        (if @selector-active?
          [:div
            [:p "Select product to add"]
            [:ul {:class "products-list"}
              (for [product @state/products-not-in-active-list]
                [:li {:key (str "prod-sel-" (:id product))
                      :on-click #(do
                                   (dispatch! :add-product-to-list {:list-id (:id @state/active-list)
                                                                    :product product})
                                   (reset! selector-active? false))}
                     (:title product)])]]
          [:button {:class "btn-action"
                    :on-click #(reset! selector-active? true)} "Add Product"])])))
{% endhighlight %}

This is an example of a "Form 2" Reagent component - it sets up local state as
a let form wrapping a function that returns a component. It sounds a little
complicated, but this is a really nice way to establish local state without
baring React's state management facilities. Since the `selector-active?` var is
bound to a Reagent atom, the component will automatically re-render whenever
the value of that atom changes.

This component also makes use of some of the `reaction`s from the state namespace.
For instance, the `products-not-in-active-list` reaction is calculated from both
the list of products and the current list. Whenever one of these underlying pieces
of data is changed, the reaction is updated so that it returns only the products
that are not already in the active list, and the UI component is then updated. As
someone who is used to using React from JavaScript, this **definitely** feels like
an improvement!

### Integrating with the server

Finally, we're ready to hook up the UI to the server. 

#### Transit support

One of the nice things about writing Clojure on the front-end as well as the
back-end is that you can transfer the data with an encoding that is handled the
same at both ends. The simplest option is to pack and unpack data structures
with [EDN](https://github.com/edn-format/edn), Clojure's native data format.
However, in the interest of performance and compatibility with other languages,
we'll use [Transit](https://github.com/cognitect/transit-format), which can
encode the same semantics as EDN but in a more efficient format for parsing
across languages. Transit, being a product of Cognitect, enjoys great support
in both Clojure and ClojureScript libraries.

We'll add the transit dependency to our `project.clj`:

{% highlight clojure %}
:dependencies [...
               [com.cognitect/transit-cljs "0.8.220"]]
{% endhighlight %}

#### API Namespace

We'll keep all of our server integration code in a `restful-cljs.api` namespace
that will contain the ajax code as well as a few helper functions for building
and parsing URLs.

{% highlight clojure %}
(ns restful-cljs.api
  (:require [clojure.string :refer [join]]
            [cognitect.transit :as t]
            [goog.events :as events])
  (:import [goog.net XhrIo EventType]))

(def json-reader (t/reader :json))
(def json-writer (t/writer :json))

(def http-methods
  {:get "GET"
   :post "POST"
   :put "PUT"
   :delete "DELETE"})

(defn with-headers [extra-headers headers-map]
  (clj->js
    (merge headers-map (or extra-headers {}))))

(defn append-qparams [url params]
  (if params
    (str url "?" (join "&" (for [[k v] params]
                            (str (name k) "=" v))))
    url))

(defn parse-response-body [xhr]
  (if-let [resp (.getResponseText xhr)]
    (if (= "" resp) "" (t/read json-reader resp))
    ""))

(defn parse-response-headers [xhr]
  (js->clj (.getResponseHeaders xhr)))

(defn json-request [{:keys [method url data headers on-success on-error]}]
  (let [xhr (XhrIo.)
        method (get http-methods method "GET")]
    (events/listen xhr EventType.SUCCESS
      (fn [e]
        (when on-success
          (on-success (parse-response-body xhr)
                      (parse-response-headers xhr)))))
    (events/listen xhr EventType.ERROR
      (fn [e]
        (when on-error
          (on-error (parse-response-body xhr)
                    (parse-response-headers xhr)))))
    (. xhr
      (send (if (= method "GET") (append-qparams url data) url)
            method
            (when data (t/write json-writer data))
            (with-headers headers {"Content-Type" "application/json"
                                   "Accept" "application/json"})))))
;; ...
{% endhighlight %}

TODO:

1. [X] Add Transit 
2. [ ] Add API namespace
3. [ ] Add API calls to actions
4. [ ] Note about running figwheel locally to test but using vm for build

If you have cloned the GitHub repo, you can fire up a VM that will server both
the API and front-end that you can access at: `http://192.168.33.10` (or whatever
you set the private network IP to in your `Vagrantfile`).

### Resources:

- [The Functional Final Frontier (video)](http://www.infoq.com/presentations/om-clojurescript-facebook-react)
- [re-frame - FRP pattern for Reagent apps](https://github.com/Day8/re-frame) - I would highly recommend this for
writing real apps with Reagent

### Go To

- [Part 1: Setup](/restful-clojure/2014/02/16/writing-a-restful-web-service-in-clojure-part-1-setup/)
- [Part 2: Getting a web server up and running](/restful-clojure/2014/02/19/getting-a-web-server-up-and-running-with-compojure-restful-clojure-part-2/)
- [Part 3: Building out the web service](/restful-clojure/2014/03/01/building-out-the-web-service-restful-clojure-part-3/)
- [Part 4: Securing the service](/restful-clojure/2015/03/13/securing-service-restful-clojure-part-4/)
- Part 5: Constructing a ClojureScript UI
