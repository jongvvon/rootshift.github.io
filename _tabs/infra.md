---
layout: page
title: 인프라 실무
icon: fas fa-server
order: 1
---

{% assign posts = site.posts | where_exp: "post", "post.categories contains '인프라 실무'" %}
{% for post in posts %}
- [{{ post.title }}]({{ post.url }})
{% endfor %}
