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

<h4>Ordered Article Tag List</h4>

{% assign sorted_tags = site.tags | sort %}

{% for tag in sorted_tags %}
  <p style="margin: 0.5rem 0 0.2rem 0; font-size: 0.9rem;">
    <strong>#{{ tag[0] }}</strong>:
    {% for post in tag[1] %}
      <a href="{{ post.url }}" style="text-decoration: none; color: #2b6cb0; margin-left: 8px;">
        {{ post.title }} <small style="color: #718096;">({{ post.date | date: "%b '%y" }})</small>
      </a>{% unless forloop.last %}, {% endunless %}
    {% endfor %}
  </p>
{% endfor %}

[Tags Graph]({% link tags-graph.html %})

[![Hits](https://hits.sh/pauljnav.github.io.svg)](https://hits.sh/pauljnav.github.io/)
