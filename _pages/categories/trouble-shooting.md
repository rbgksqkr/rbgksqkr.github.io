---
title: "프로젝트를 진행하며 겪은 문제 상황과 해결 방법을 공유합니다."
layout: archive
permalink: categories/trouble-shooting
author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories['trouble-shooting']%}
{% for post in posts %}
{% include archive-single.html type=page.entries_layout %}
{% endfor %}
