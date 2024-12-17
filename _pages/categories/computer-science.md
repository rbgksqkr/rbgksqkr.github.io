---
title: "CS 개념을 정리하며 기술 면접 준비"
layout: archive
permalink: categories/computer-science
author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories['computer-science']%}
{% for post in posts %}
{% include archive-single.html type=page.entries_layout %}
{% endfor %}
