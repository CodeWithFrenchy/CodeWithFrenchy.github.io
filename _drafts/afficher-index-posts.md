---
title: Astuce pour afficher tous les index des publications
date: 2022-06-05 19:58:00 -0400
categories: []
tags: []
---

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

Voir ce [lien](https://jekyllrb.com/docs/posts/#displaying-an-index-of-posts).
