---
layout: post
title: "Securing the service | RESTful Clojure, Part 4"
description: "Protect your new service's resource with authorization and authentication"
category: restful-clojure
tags: ["clojure", "REST", "authorization", "authentication", "tutorial"]
---

*Drum roll, please...* After more than a year, I finally have some time to pick
up this blog series on creating RESTful data APIs with Clojure. It is amazing to me
how much the Clojure ecosystem has changed (for the better) even in the past year.
One of these changes has been the release of the the [Buddy security library](https://github.com/funcool/buddy)
Okay, it has been around for over a year, but in the past few months or so, Buddy
has become the de-facto library for Clojure web app security.

Originally, I had planned on using [Friend](https://github.com/cemerick/friend) -
which I still think is a great library - but Buddy is far more flexible and jives
well with the Clojure community's focus of **building amazing things from small, composable parts**.
Buddy consists of modules for crypto, auth/auth, password hashing, and
message signing, which may be used together or a la carte. For our shopping list
API, we will use the auth and password hashing functionality of Buddy. We will
only be scratching the surface of both web security in general and the Buddy library
specifically, but I have listed some [Resources](#resources) at the end of this article
that are definitely worth a read/watch.

## Security Architecture

There are a number of options for securing a web service or data API. One of the
main considerations is whether the API will be consumed by other back-end services
or by front-end applications. In our case, the API will be consumed by a
JavaScript application directly, so we will provide a mechanism that will be
simple for that sort of client to consume.

We will introduce two different user types - `:user` and `:admin` - who will each
have different permissions in the system. Admins should be able to do everything
that regular users can as well as a few more privileged actions. Users will authenticate
with the system by providing an _auth token_ in an `Authorization` HTTP header,
which the server will verify to determine whether the auth token is valid
and which user it is associated with.

Note that for an application with higher security requirements, this approach
would probably not be sufficient because if a hacker got the auth token, they
could have access to the system without even knowing the identity of the user
that they were impersonating. However, as long as the initial token request _as
well as every single request containing the token_ happens over HTTPS, we don't
need to be too worried.

If a logged-out client requests a resource that requires authentication, or if a
logged-in client an requests a resource that they are not authorized for, an
`HTTP 401 Unauthorized` response should be returned. Eventually, we would probably
want to return the 401 only for unauthenticated requests and use an
`HTTP 403  Forbidden` instead for requests that to not have adequate authorization,
but to make things simple for this tutorial, we'll stick with returning the 401
in both cases.

## Code, glorious code

With those high-level requirements outlined, let's start turning them into tests.
The previous tutorials were pretty code-heavy, but by now I assume that you know
how the project is organized, so I'll only include the most pertinent portions of
code. The full source code is available on [the companion repo](https://github.com/kendru/restful-clojure).

First, we'll implement password hashing for users. We want to be able to validate
a user's password without storing their password in the database in plain text.
Buddy's [hashers](http://funcool.github.io/buddy-hashers/latest/) module is
perfect for this.

### Passwords for users

{% highlight clojure %}
; test/restful_clojure/users_test.clj
; ...
(deftest authorize-users
  (let [user (users/create {:name "Sly" :email "sly@falilystone.com" :password "s3cr3t"})
        user-id (:id user)]
    (testing "Accepts the correct password"
      (is (users/password-matches? user-id "s3cr3t")))

    (testing "Rejects incorrect passwords"
      (is (not (users/password-matches? user-id "not_my_password"))))))
; ...
{% endhighlight %}

Before we can write the code to get these tests passing, we need to create a
database migration adding a `password_digest` column to the `users` table. We also
need to add the buddy-hashers dependency to our `project.clj`. Once those steps
are complete, we're ready to update the `restful-clojure.models.users`
namespace.

{% highlight clojure %}
; src/restful_clojure/models/users.clj
; ...
(defn find-by [field value]
  (some-> (select* e/users)
          (where {field value})
          (limit 1)
          select
          first
          (dissoc :password_digest)))
; ...
(defn create [user]
  (-> (insert* e/users)
      (values (-> user
                  (assoc :password_digest (hashers/encrypt (:password user)))
                  (dissoc :password)))
      insert
      (dissoc :password_digest)))
; ...
(defn password-matches?
  "Check to see if the password given matches the digest of the user's saved password"
  [id password]
  (some-> (select* e/users)
            (fields :password_digest)
            (where {:id id})
            select
            first
            :password_digest
            (->> (hashers/check password))))
{% endhighlight %}

Most of this is pretty standard stuff for a web app, but notice that we are
taking care that the password is hashed and stored in the database but that even
the hash is never exposed to the client. When you run the tests now, you'll also
notice that they are much slower. This is due to the fact that the bcrypt algorithm
that Buddy uses by default to hash passwords is slow. Slowness is a good thing
when it comes to password hashing because the slower a hash algorithm is, the
less effective it renders brute-force attacks.

### Auth tokens

The next order of business is allowing users to supply their user id and password
and get an auth token that is valid for some specific amount of time. This time
I'll skip the tests because they are not very interesting.

We'll need to create a new migration that will create an `auth_tokens` table:

{% highlight sql %}
CREATE TABLE auth_tokens (
    id VARCHAR(36) PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX ON auth_tokens (id, created_at DESC);
{% endhighlight %}

While we're at it, let's go ahead and create another migration adding a `level`
column to the users table to identify a user a being a "user" or "admin".

{% highlight sql %}
ALTER TABLE users ADD COLUMN level VARCHAR(12) NOT NULL default 'user';
{% endhighlight %}

We will bundle both authorization and authentication together into one "auth"
namespace. I'll show you the full code below, and then we'll walk through what it
does. This is probably the longest code sample in the tutorial, but it contains
most of the internals of auth, so try to follow along.

{% highlight clojure %}
; src/restful_clojure/auth.clj
(ns restful-clojure.auth
  (:use korma.core)
  (:require [restful-clojure.entities :as e]
            [restful-clojure.models.users :as users]
            [buddy.auth.backends.token :refer [token-backend]]
            [buddy.auth.accessrules :refer [success error]]
            [buddy.auth :refer [authenticated?]]
            [crypto.random :refer [base64]]))

(defn gen-session-id [] (base64 32))

(defn make-token!
  "Creates an auth token in the database for the given user and puts it in the database"
  [user-id]
  (let [token (gen-session-id)]
    (insert e/auth-tokens
      (values {:id token
               :user_id user-id}))
    token))

(defn authenticate-token
  "Validates a token, returning the id of the associated user when valid and nil otherwise"
  [req token]
  (let [sql (str "SELECT user_id "
                 "FROM auth_tokens "
                 "WHERE id = ? "
                 "AND created_at > current_timestamp - interval '6 hours'")]
    (some-> (exec-raw [sql [token]] :results)
            first
            :user_id
            users/find-by-id)))

(defn unauthorized-handler [req msg]
  {:status 401
   :body {:status :error
          :message (or msg "User not authorized")}})

;; Looks for an "Authorization" header with a value of "Token XXX"
;; where "XXX" is some valid token.
(def auth-backend (token-backend {:authfn authenticate-token
                                  :unauthorized-handler unauthorized-handler}))

;; Map of actions to the set of user types authorized to perform that action
(def permissions
  {"manage-lists"    #{:restful-clojure.models.users/user}
   "manage-products" #{:restful-clojure.models.users/admin}
   "manage-users"    #{:restful-clojure.models.users/admin}})

;;; Below are the handlers that Buddy will use for various authorization
;;; requirements the authenticated-user function determines whether a session
;;; token has been resolved to a valid user session, and the other functions
;;; take some argument and _return_ a handler that determines whether the
;;; user is authorized for some particular scenario. See handler.clj for usage.

(defn authenticated-user [req]
  (if (authenticated? req)
    true
    (error "User must be authenticated")))

;; Assumes that a check for authorization has already been performed
(defn user-can
  "Given a particular action that the authenticated user desires to perform,
  return a handler determining if their user level is authorized to perform
  that action."
  [action]
  (fn [req]
    (let [user-level (get-in req [:identity :level])
          required-levels (get permissions action #{})]
      (if (some #(isa? user-level %) required-levels)
        (success)
        (error (str "User of level " (name user-level) " is not authorized for action " (name action)))))))

(defn user-isa
  "Return a handler that determines whenther the authenticated user is of a
  specific level OR any derived level."
  [level]
  (fn [req]
    (if (isa? (get-in req [:identity :level]) level)
      (success)
      (error (str "User is not a(n) " (name level))))))

(defn user-has-id
  "Return a handler that determines whether the authenticated user has a given ID.
  This is useful, for example, to determine if the user is the owner of the requested
  resource."
  [id]
  (fn [req]
    (if (= id (get-in req [:identity :id]))
      (success)
      (error (str "User does not have id given")))))
{% endhighlight %}

First, we create a utility function for generating session identifiers, which
are cryptographically strong random bits that are base-64 encoded. While it would
be easier to use something like a UUID here, that would create a guessable
session identifier, making it much easier for a hacker to exploit the system. I
am using James Reeve's [crypto-random](https://github.com/weavejester/crypto-random)
library here, but any strong random generator is okay here (even pulling bytes
off of `/dev/urandom`).

Next, the `make-token!` function simple creates a new session in the database
associated with the given user id.

We then create a Buddy token-based authentication backend, that will hook into
our Ring middleware and look for a "Authorization" HTTP header, extracting the
token from that. If a valid token is found, Buddy will associate the returned
user in the Ring request map, which we can use to either require that the user is
logged-in or that they have a specific user level. The Buddy middleware will call
the `authenticate-token` function with the Ring request map and the token found
in the "Authorization" header and will expect a user if the token was valid and
`nil` otherwise.

Finally, we create a simple permission structure such that users can only manage
lists (create, add/remove products, etc.), and admins can manage products and
users in addition to everything that users can do. In order to create the user
hierarchy such that admins will inherit all user privileges, we will create an
ad-hoc hierarchy in the users namespace. We also need to make a few changes
additional changes to accommodate the addition of user levels. Most of the relevant
code from the users namespace is below.

{% highlight clojure %}
; src/restful_clojure/models/users.clj
; ...
(def user-levels
  {"user" ::user
   "admin" ::admin})
(derive ::admin ::user)

(defn- with-kw-level [user]
  (assoc user :level
              (get user-levels (:level user) ::user)))

(defn- with-str-level [user]
  (assoc user :level (if-let [level (:level user)]
                       (name level)
                       "user")))
; ...
(defn find-by [field value]
  (some-> (select* e/users)
          (where {field value})
          (limit 1)
          select
          first
          (dissoc :password_digest)
          with-kw-level))
; ...
(defn create [user]
  (-> (insert* e/users)
      (values (-> user
                  (assoc :password_digest (hashers/encrypt (:password user)))
                  with-str-level
                  (dissoc :password)))
      insert
      (dissoc :password_digest)
      with-kw-level))
; ...
{% endhighlight %}

The two changes here are that we create a hierarchy of `::user`s and `::admin`s
and that we convert between string representations for storage and keyword
representations for programmatic operation. Another minor detail is that we
ensure users with no level specified are always stored with the "user" type.

Finally, we make the necessary changes in our handlers to restrict endpoints to
logged-in users/users with the appropriate privileges. This is a good time to
refactor the handler namespace to extract the business logic for each route to
its own function. As the app grows, these will probably move into other namespaces,
so for now having the business logic isolated from the routing logic will come in
handy. With the routes cleaned up, it is now easier to see how the various
authorization rules play out with our application routes.

{% highlight clojure %}
; src/restful_clojure/models/users.clj
; ...
(defroutes app-routes
  ;; USERS
  (context "/users" []
    (GET "/" [] (-> get-users
                    (restrict {:handler {:and [authenticated-user
                                               (user-can "manage-users")]}
                               :on-error unauthorized-handler})))
    (POST "/" [] create-user)
    (context "/:id" [id]
      (restrict
        (routes
          (GET "/" [] find-user)
          (GET "/lists" [] lists-for-user))
        {:handler {:and [authenticated-user
                         {:or [(user-can "manage-users")
                               (user-has-id (read-string id))]}]}
         :on-error unauthorized-handler}))
    (DELETE "/:id" [id]
        (-> delete-user
            (restrict {:handler {:and [authenticated-user
                                 (user-can "manage-users")]}
                       :on-error unauthorized-handler}))))

  (POST "/sessions" { {:keys [user-id password]} :body}
    (if (users/password-matches? user-id password)
      {:status 201
       :body {:auth-token (make-token! user-id)}}
      {:status 409
       :body {:status "error"
              :message "invalid username or password"}}))

  ;; LISTS
  (context "/lists" []
    (GET "/" []
        (-> get-lists
            (restrict {:handler {:and [authenticated-user
                                       (user-isa :restful-clojure.models.users/admin)]}
                       :on-error unauthorized-handler})))
    (POST "/" [] (-> create-list
                     (restrict {:handler {:and [authenticated-user
                                                (user-can "manage-lists")]}
                                :on-error unauthorized-handler})))
    (context "/:id" [id]
      (let [owner-id (get (lists/find-by-id (read-string id)) :user_id)]
        (restrict
          (routes
            (GET "/" [] find-list)
            (PUT "/" [] update-list)
            (DELETE "/" [] delete-list))
          {:handler {:and [authenticated-user
                           {:or [(user-can "manage-lists")
                                 (user-has-id owner-id)]}]}
           :on-error unauthorized-handler}))))

  ;; PRODUCTS
  (context "/products" []
    (restrict
      (routes
        (GET "/" [] get-products)
        (POST "/" [] create-product))
      {:handler {:and [authenticated-user (user-can "manage-products")]}
       :on-error unauthorized-handler}))

  (route/not-found (response {:message "Page not found"})))
{% endhighlight %}

I admit that some of the nesting of routes within contexts can get a little ugly.
Many developers prefer to write "flat" routes with a little more duplication,
which is admittedly easier to read, but it can become more difficult to maintain
with code duplicated between a number of route definitions.

Note, however, how nicely we can express our auth requirements, as in the following
example extracted from the routes above:

{% highlight clojure %}
(restrict
  (routes
    (GET "/" [] find-list)
    (PUT "/" [] update-list)
    (DELETE "/" [] delete-list))
  {:handler {:and [authenticated-user
                   {:or [(user-can "manage-lists")
                         (user-has-id owner-id)]}]}
   :on-error unauthorized-handler}))
{% endhighlight %}

This very declaratively expresses that, "For finding, updating, and deleting
a specific list, we need the user to be authenticated and either be able to
manage lists or be the owner of the specific list". This, I think, is where
Clojure really shines - declarative APIs expressed as data. Did you notice that
we lay out our auth rules as some nested maps and vectors? I'll take that over
an imperative auth system any day!

## In Conclusion

I have only briefly glossed over a couple of the many security concerns that
web apps face. Please know that this is only the tip of the iceberg and that
there is much more to building a "bulletproof" web service that what was
covered here. That said, the authorization and authentication practices in
this tutorial should pretty much cover what you'll need to build into a typical
web service or application of this scale.

If you do not have the code for this tutorial, I'd recommend checking it out
from [its GitHub repo](https://github.com/kendru/restful-clojure) and playing
around with it. If you have been walking through these tutorials from
[part 1](/restful-clojure/2014/02/16/writing-a-restful-web-service-in-clojure-part-1-setup/),
then you should now have a complete, albeit simple, API server that you can deploy
and build a front-end app on!

Next up, we'll build a simple front-end app to consume our web service, then finally,
we will deploy it to a DigitalOcean droplet. Stay tuned!

### Resources:

- [Securing Clojure Microservices using buddy - Part 1: Creating Auth Tokens](http://rundis.github.io/blog/2015/buddy_auth_part1.html)
- [(Video) Aaron Bedra - clojure.web/with-security](https://youtu.be/CBL59w7fXw4)
- [OWASP Top 10](https://www.owasp.org/index.php/Category:OWASP_Top_Ten_Project#tab=OWASP_Top_10_for_2013)

### Go To

- [Part 1: Setup](/restful-clojure/2014/02/16/writing-a-restful-web-service-in-clojure-part-1-setup/)
- [Part 2: Getting a web server up and running](/restful-clojure/2014/02/19/getting-a-web-server-up-and-running-with-compojure-restful-clojure-part-2/)
- [Part 3: Building out the web service](/restful-clojure/2014/03/01/building-out-the-web-service-restful-clojure-part-3/)
- Part 4: Securing the service
