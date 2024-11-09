---
title: "알고리즘 풀이 후 풀이 기법 정리 - 문법"
layout: archive
permalink: categories/algorithm/basic
author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories['기초']%}
{% for post in posts %}
{% include archive-single.html type=page.entries_layout %}
{% endfor %}
