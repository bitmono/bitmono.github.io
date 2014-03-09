---
layout: index
title: Bitmono
---

<ul>
  {% for post in site.posts %}
    <li>
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      <p class="meta">Posted {{ post.date | date_to_string }} by {{ post.post_author }}</p>

      <p>{{ post.excerpt }}</p>
    </li>
  {% endfor %}
</ul>
