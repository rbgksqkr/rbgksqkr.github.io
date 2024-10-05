---
title: "회고"
layout: archive
permalink: categories/retrospect
author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories['retrospect']%}
{% for post in posts %}
{% include archive-single.html type=page.entries_layout %}
{% endfor %}
