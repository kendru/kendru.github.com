---
layout: post
title: "First thoughts on Elixir"
description: "Random thoughts on my first few days with Elixir"
category: Elixir
tags: ["programming", "elixir"]
---

For the past few days, I have been playing around with Elixir to build
an API gateway for WP-Pulsar. While I am far from an Elixir expert (I'd
barely even consider myself and Elixir novice yet), I have really started
to fall in love with the language!

I tried learning Erlang a few years ago, and while it was an interesting
experience, the syntax was a bit of a turn-off, and I could not find a lot
of information available on building web applications with Erlang - which is
what I was working on at the time. My interests turned more to Clojure and
the JVM at the time, and I pretty much forgot about Erlang.

Now that I have decided to learn Elixir, I have to say that it is probably
the most _fun_ that I have had programming since I moved from PHP to Ruby.
Even in a short time, it has started to change the way that I think about
building systems in a functional language.

### Syntax

First, there is the syntax. Largely inspired by Ruby, the code is delimited
with `do` ... `end` blocks, like so:

{% highlight elixir %}
def greet(name) do
  "Hello, #{name}"
end
{% endhighlight %}

Apparently, this is just syntactic sugar for a
[keyword list](https://elixir-lang.org/getting-started/keywords-and-maps.html),
which is a list of 2-tuples whose first element is an atom (basically a symbol
that simply evaluates to itself), so that same function can be defined as:

{% highlight elixir %}
def greet(name), [{:do, "Hello, #{name}"}]
{% endhighlight %}

Sweet! That's definitely uglier, but it hints at the fact that code is _almost_
data. What looked like a concrete syntactic construce is just a nested data
structure. I have not had a chance to work with Elixir's macro system yet, but I
am guessing that this sort of almost-homoiconic structure is what enables
these macros.

For the most part, the syntax reminds me of Ruby, but I can remember a number
of times thinking, "Oh, I remember *that* from Erlang", such as pattern matching
a list: `[ head | tail ] = my_list`. Also, the pipeline operator (`|>`) is very
similar to Clojure's threading macro (`->`). Overall, the language has a pretty
friendly syntax, and it felt familiar to my, having worked with both Ruby and
some functional languages in the past.

### The Best Parts

The nice syntax meant that there was a low barrier to entry, but it's not what is
winning me over. The things that make Elixir stand out are its advanced pattern
matching, functions with multiple parameter lists and bodies, the actor model, and
OTP.


#### Pattern matching

First off, Elixir has an `if/else` construct, but I haven't used it. Instead, I am
relying on pattern matching and functions with multiple parameter lists and bodies.
For example, I might write something like the following in JavaScript:

{% highlight javascript %}
const jobsUrl = (type) => {
  if (type === 'db') {
    return apiUrl('db-jobs');
  }
  return apiUrl('file-jobs');
};
{% endhighlight %}

In Elixir, I would just write a function definition for each possible branch:

{% highlight elixir %}
def jobs_url(:file), do: api_url "/file-jobs"
def jobs_url(:db),   do: api_url "/db-jobs"
{% endhighlight %}

There is one definition of the function when it is called with `:file` and another
when it is called with `:db`. In my opinion, this is much clearer and helps avoid
incidental complexity. You can get much more advanced with pattern matching and
even add guards to restrict the match to some narrower criteria. For example,
the following code checks some HTTP response and determines whether it was
successful.

{% highlight elixir %}
# Succeeded with a normal status code
def resp_status({:ok, %{status_code: status_code}}) when status_code in 200..299 do
  :ok
end

# Succeeded, but not with a status we expected
def resp_status({:ok, _}) do
  :error
end

# Failed with an error
def resp_status({:error, _}) do
  :error
end
{% endhighlight %}

I am still getting used to thinking in terms of pattern matching rather than
imperative branching, but so far it has made for much more maintainable code
(i.e. I can still understand what I wrote the following day).

#### Actor model and OTP

First off, I have not had a chance to do a ton of concurrent programming with
Elixir, so most of my exposure to processes and OTP have been through trivial
examples like spinnng up a bunch of processes and passing messages around in
a ring. Still, even with toy examples, I'm beginning to see the power that
Elixir's (and Erlang's) actor model could bring to an application like the
API gateway/reverse proxy I am building.

In Elixir, we run things concurrently by running them in their own `process`
and exchanging messages back and forth. Each process gets its own heap (which
is garbage-collected independently) and scheduled by the Erlang VM. However,
unlike OS processes, these virtual processes are super lightweight (I just
created 1,000,000 of them as a test on my desktop without any problems). I am
finding that I am thinking of my code in terms of a society and imagining the
processes as little people who each have some role in moving that society to
some goal. Maybe that's a common way to feel when encountering this model of
programing, but it has been a log of fun.

On top of the actor-based concurrency model that is enabled by lightweight
processes, Elixir can take advantage of the Erlang ecosystem, including OTP.
As far as I can tell OTP is both a library and a set of conventions for
building concurrent, scalable applications. It includes `GenServer`, which is
a `behaviour` that takes care of some of the boilerplate for defining processes
that act as servers and respond to certain types of requests. Being this new
to Elixir/Erlang/OTP, it seems like a behaviour is essentially a convention or
contract that a process must follow that provides a well-known interface for
other processes to use. I'm sure I'm missing subtleties here, and I could be
downright wrong, but that is my current take on it.

OTP also has the concept of supervisors, which are processes that are solely
responsible for monitoring other processes and deciding what to do when a
process under their watch exits. One of the cool things is that supervisors
can manage other supervisors, so you can form a hierarchy of supervised
processes. Like I mentioned before, this makes the application feel much more
organic and like an analogy for a human society.

I hear that there is much more to OTP, including a
[key/value store](https://elixir-lang.org/getting-started/mix-otp/ets.html)
and a [fully-fleged database](http://erlang.org/doc/apps/mnesia/Mnesia_overview.html),
but I'm still trying to get the basics down.

Hopefully this post helped convey some of the things that make Elixir a
worthwhile language to learn. I hope to be able to post some more focused,
in-depth thoughts as I dig deeper into the language!

### Key Takeaways:

- The Ruby-like syntax makes Elixir easy to learn
- Pattern matching is really cool and eliminates the need for most conditionals
- Similarly, splitting a function into multiple bodies is a clean way to
  refactor code
- OTP is huge but amazing
- Processes are really, really, really cheap

