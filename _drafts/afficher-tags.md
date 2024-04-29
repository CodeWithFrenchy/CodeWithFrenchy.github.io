---
title: Astuce pour voir tous les tags
date: 2022-06-05 19:58:00 -0400
categories: []
tags: []
---

{% for tag in site.tags %}
  <h3>{{ tag[0] }}</h3>
  <ul>
    {% for post in tag[1] %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}

Voir ce [lien](https://jekyllrb.com/docs/posts/#tags).
