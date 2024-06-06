---
title: Dustin Dortch
---

{% for post in site.posts %}
  <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
  <p>{{ post.excerpt }}</p>
  <p><a href="{{ post.url }}">Read more</a></p>
{% endfor %}
