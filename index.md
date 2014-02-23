---
layout: page
title: kendru/write
tagline: coding, startups, and software engineering
description: a blog on software engineering (mainly Clojure at the moment) and startups
---
{% include JB/setup %}

## latest posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

