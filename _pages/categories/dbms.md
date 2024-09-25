---
layout: archive
permalink: dbms
title: "DBMS"

author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories.dbms %}
{% for post in posts %}
  {% include custom-archive-single.html type=entries_layout %}
{% endfor %}
