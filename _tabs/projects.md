---
layout: page
title: 사이드 프로젝트
icon: fas fa-rocket
order: 4
---

{% assign posts = site.posts | where_exp: "post", "post.categories contains '사이드 프로젝트'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}
