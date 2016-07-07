---
layout: page
title: Posts by category
permalink: /categories/
---

{% capture catz %}
  {% for cat in site.categories %}
    {{ cat[0] }}
  {% endfor %}
{% endcapture %}
{% assign sortedcatz = catz | split:' ' | sort %}

{% for cat in sortedcatz %}
  <h2 id="{{ cat }}">{{ cat }}</h2>
  <ul>
  {% for post in site.categories[cat] %}
    <li><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
  </ul>
{% endfor %}
