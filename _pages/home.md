---
layout:     page
title:      Home
permalink:  /home
nav_order:  1
pagination: 
  enabled: true
---

<div class="posts">
  {% for post in site.posts %}
  <div class="post">
    <h1 class="post-title">
      <a href="{{ post.url }}">
        {{ post.title }}
      </a>
    </h1>
    <span class="post-date">{{ post.date | date_to_string }}</span>
    {{ post.excerpt }}
  </div>
  {% endfor %}
</div>