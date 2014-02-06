---
layout: page
title: (write andrew blog)
tagline: Coding, startups, and software engineering
---
{% include JB/setup %}

## latest posts

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

