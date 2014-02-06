---
layout: post
title: "Simple is better"
description: "When designing a new product or software feature, choose the
simplest implementation that works"
category: "Software Engineering"
tags: ["simplity", "design"]
---
{% include JB/setup %}

## Why simple?

When I'm committing to a project, I have a general rule of thumb that I like to
remove more lines of code than I create. Of course this does not always happen,
but I get a warm, fuzzy feeling from knowing that I implemented a feature by
_removing_ code. Why? Because when there is less code, there is generally less
complexity, and when there is less complexity, the code is easier to grok.

Really, the issue is not that _smaller_ code is better, per se, but _simpler_
code is. In fact, *the best code for any given problem is the simplest possible
code that actually solves the problem*. Of course, there are some problems that
are inherently complex, and these will require a complex solution. However,
even among complex solutions, there are solutions that are simpler than others.

Let's take a naive example: your task is to write a program that prints the
words "Hello World" to the screen? (sarcasm alert)  Well, if we need to print
some words, then we naturally need an abstraction for words, right? We'll
certainly also need a separate entity for printing the words to the screen.
But then, we'll probably also want to do other things with words than print
them to the screen - after all, our next assignment will probably be to write
them to a file or a printer. Looks like we need an interface...

Before long, this is what we end up with:

{% highlight php %}
<?php
class MyString
{
   private $str;

   public function __construct($str)
   {
      $this->str = $str;
   }

   public function unpack()
   {
      return $this->str;
   }
}

class MyStringHolder
{
   private $strings = [];

   public function addString(MyString $str)
   {
      $unpacked = $string->unpack();
      if (empty($unpacked)) return;
      $this->strings[] = $string;
   }

   public function getStrings()
   {
      return $this->strings;
   }
}

interface MyStringHolderViewInterface
{
   public function render(MyStringHolder $holder, array $options);
}

class StdOutPrinterStringHolderView implements MyStringHolderViewInterface
{
   public function render(MyStringHolder $holder,
                          array $options = ['delimeter' => ' '])
   {
      $delimeter = isset($options['delimeter']) ? $options['delimeter'] : ' ';
      $finalString = '';
      foreach($holder->getStrings() as $i => $packedString) {
         $finalString .= ($i >= 1) ? $delimeter : '';
         $finalString .= $packedString->unpack();
      }

      $this->doRender($finalString);
   }

   private function doRender($string)
   {
      if (!empty($string)) {
         print($string);
      }
   }
}

$hello = new MyString("Hello");
$world = new MyString("World");
$helloWorldHolder = new MyStringHolder();
$helloWorldHolder->addString($hello);
$helloWorldHolder->addString($world);
$printer = new StdOutPrinterStringHolderView();
$printer->render($helloWorldHolder);
{% endhighlight %}

I _sincerely_ hope that you didn't actually read all of that. For the poor
souls who did, how apparent was it from reading the code that it prints
"Hello World" to the screen?

Oh, but it doesn't actually do that - did I metion that there's a bug in the
code? Yep. It actually doesn't print anything. I don't know about you, but I don't
really want to track down that bug right now. (If you're curious where the bug
is, look at the parameters for MyStringHolder::addString).

Maybe an implementation like this would eventually be necessary for your
project. If that's the case, only resort to the complex implementation when
necessary. You don't need the additional layers of abstraction today, so leave
them out.

## What is simple?

Simplicity in software is not easy to define. Short code is not always
simple. Pattern-centric code is not always simple. No specific paradigm of code
- OO, functional, imperative, etc. - is always simple. However, I believe that
the essence of simplicity boils down to this:

*Simple code is code that looks like the problem you are trying to solve.*

To borrow the "Hello World" example from above, this is the simplest
implementation that I could think of:
{% highlight php %}
print "Hello World";
{% endhighlight %}

This solution has a verb, "print", and a noun, "Hello World". The vocabulary
for the solution comes straight from the problem itself. Since this code
performs the designated task with the least amount of abstraction and the least
additional vocabulary, it is truly simple.

While this example is overly simplistic, and the actual problems that we as
software engineers face are vastly more intricate, we should take away the idea
that _lean_, _concise_ code is always preferrable to overly-designed,
overly-abstracted code.

## Is complex ever okay?

Yes, but only when we choose the simplest possible solution among many complex
solutions. Some problem domains contain so many concepts and so many moving
pieces that we can only simplify to a certain extent - and this is not a bad
thing. Inherent complexity is a fact that we must live with, but that is not an
excuse for introducing additional complexity that is not warrented. Again,
always start with the most naive solution that actually works. You will
probably have to refactor it eventually, but you may not, and you will not be
left with a ton of kludge that you never end up using.

Again, this is a very simplistic look at what simple code is and why we should
strive for simplicity, but I hope it was helpful!

