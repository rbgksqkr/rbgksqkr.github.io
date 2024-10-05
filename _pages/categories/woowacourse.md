---
title: "우아한테크코스 6기 활동"
layout: archive
permalink: categories/woowacourse
author_profile: true
sidebar:
  nav: "docs"
---

{% assign posts = site.categories['woowacourse']%}
{% for post in posts %}
{% include archive-single.html type=page.entries_layout %}
{% endfor %}
