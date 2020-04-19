---
title: Articles 
layout: collection
permalink: /articles/
collection: articles 
entries_layout: grid
classes: wide
last_modified_at: 2020-03-23T14:07:54-04:00
toc: true
---

{% for article in site.articles limit: 5 %}
  {% include archive-single.html %}
{% endfor %}

