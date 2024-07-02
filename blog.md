---
layout: default
title: Blog
permalink: /blog/
---

## Blog
Welcome to my blog! Here are my latest posts:

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
      <p>{{ post.excerpt }}</p>
    </li>
  {% endfor %}
</ul>
