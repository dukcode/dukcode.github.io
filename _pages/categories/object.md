---
layout: archive
permalink: object
title: "OBJECT"

author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories.object %}
{% for post in posts %}
  {% include custom-archive-single.html type=entries_layout %}
{% endfor %}
