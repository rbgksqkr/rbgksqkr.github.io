---
title: "알고리즘 풀이 후 풀이 기법 정리 - 백트래킹"
layout: archive
permalink: categories/algorithm/backtracking
author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories['백트래킹']%}
{% for post in posts %}
{% include archive-single.html type=page.entries_layout %}
{% endfor %}
