---
layout: page
title: Journal
permalink: /journal
nav_order: 4
---

<ul style="list-style-type:none;">
  {% for journal in site.journals %}
    <li><a href="{{ journal.url }}">{{ journal.title }}&nbsp;{{ journal.description }}</a></li>
  {% endfor %}
</ul>