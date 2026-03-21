---
layout: page
title: DevOps 전환기
icon: fas fa-code-branch
order: 2
---

{% assign posts = site.posts | where_exp: "post", "post.categories contains 'DevOps 전환기'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}
