---
layout: post
title: "7 ideas that will take you from concept to product... fast"
description: "Go from product idea to marketable product quicker (and with fewer headaches) than your last project"
category: Startup
tags: ["lists", "startup", "design"]
---
{% include JB/setup %}

When you have a great idea for a new software product, here is the worst thing
that you can do:
1. Get really exited
2. Start coding
3. Get distracted
4. Have another idea
5. Abandon the project

And in this manner, countless might-have-been-something-great ideas have been
relegated to the recycle bin. While you can't guarantee that your idea is going
to be the next Google, you can guarantee that you at least get it to market.
Here are a few ideas that I have learned (mostly by doing the wrong thing and
realizing later how very wrong I was) that have helped keep my projects on
track.

## 1: Think simple

The simplest solution that works is the best solution. Think [Occam's
Razor](http://en.wikipedia.org/wiki/Occam's_razor) for software engineering. If
you can use Proxy, Factory, and Chain of Command to implement a feature in 400
lines, or you can implement it in 50 lines of straigntforward code... write it
in 50 lines. If you need the more complex implementation later, write it later.
Take the following two pieces of code below:

```
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
      if (empty($string)) return;
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
                          array $options = array['delimeter' => ' '])
   {
      $delimeter = isset($options['delimeter']) ? $options['delimeter'] : ' ';
      $finalString = '';
      foreach($holder->getStrings() as $i => $packedString)
      {
         $finalString .= ($i >= 1) ? $delimeter : '';
         $finalString .= $packedString->unpack();
      }
   }
```

## 2:
## 3:
## 4:
## 5:
## 6:
## 7:
## 8: Focus
And you thought there were only seven points! Number 8 here is more of a
summary point. ...

