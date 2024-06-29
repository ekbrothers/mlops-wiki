---
layout: home
title: total soccer Wiki
---

# Welcome to the total soccer Wiki!

This wiki contains information about total soccer.

## Topics

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
