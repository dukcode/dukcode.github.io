---
layout: archive
permalink: git
title: "GIT"

author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories.git %}
{% for post in posts %}
  {% include custom-archive-single.html type=entries_layout %}
{% endfor %}
