---
layout: page
title: Blog Archive
---

<ul class="toc-list">
  {% for post in site.posts %}
    <li class="toc-item">
      <a class="toc-link" href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <span class="toc-date">{{ post.date | date: "%B %d, %Y" }}</span>
    </li>
  {% endfor %}
</ul>

{% for tag in site.tags %}
  <h3>{{ tag[0] }}</h3>
  <ul>
    {% for post in tag[1] %}
      <li><a href="{{ post.url }}">{{ post.date | date: "%B %Y" }} - {{ post.title }}</a></li>
    {% endfor %}
  </ul>
{% endfor %}

[Tags Graph]({% link tags-graph.html %})

[![Hits](https://hits.sh/pauljnav.github.io.svg)](https://hits.sh/pauljnav.github.io/)
