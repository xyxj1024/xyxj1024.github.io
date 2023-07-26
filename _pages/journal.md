---
layout: page
title: Journal
permalink: /journal
nav_order: 4
---

<ul style="list-style-type:none;">
  {% for journal in site.journals %}
    <li><a href="{{ journal.url }}">{{ journal.date | date_to_string }}&nbsp;{{ journal.title }}</a></li>
  {% endfor %}
</ul>