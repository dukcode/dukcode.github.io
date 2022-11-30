---
layout: archive
permalink: cicd
title: "CI/CD"

author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories.cicd %}
{% for post in posts %}
  {% include custom-archive-single.html type=entries_layout %}
{% endfor %}
