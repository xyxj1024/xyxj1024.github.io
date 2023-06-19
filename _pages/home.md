---
layout:     page
title:      Home
permalink:  /home
nav_order:  1
pagination: 
  enabled: true
---

<!-- Html Elements for Search -->
<div id="search-container">
  <input type="text" id="search-input" placeholder="Search blog posts by title, tags, date, etc.">
  <ul id="results-container"></ul>
</div><br>

<div class="posts">
  {% for post in site.posts %}
  <br><hr>
  <div class="post">
    <h3 class="post-title">
      <a href="{{ post.url }}">
        {{ post.title }}
      </a>
    </h3>
    <span class="post-date">{{ post.date | date_to_string }}</span>
    {{ post.excerpt }}
  </div>
  {% endfor %}
</div>

<!-- Script pointing to search-script.js -->
<script type="text/javascript" src="search-script.js"></script>

<!-- Configuration -->
<script>
SimpleJekyllSearch({
  searchInput: document.getElementById('search-input'),
  resultsContainer: document.getElementById('results-container'),
  json: '/search.json'
})
</script>