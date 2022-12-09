---
layout:     page
title:      Blog Posts
permalink:  /posts
nav_order:  3
---

{% assign sorted_categories = site.categories | sort %}
{% for category in sorted_categories %}
  <h3>{{ category[0] }}</h3>
  <ul>
    {% assign sorted_posts = category[1] | sort: 'title' %}
    {% for post in sorted_posts %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}