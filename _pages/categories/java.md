---
layout: archive
permalink: java
title: "JAVA"

author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories.java %}
{% for post in posts %}
  {% include custom-archive-single.html type=entries_layout %}
{% endfor %}
