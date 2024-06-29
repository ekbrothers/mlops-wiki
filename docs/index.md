---
layout: home
title: Civil War Era Coffee Consumption
---

# Welcome to the Civil War Era Coffee Consumption!

This wiki contains information about coffee in the civil war.

## Topics

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url | relative_url }})
{% endfor %}
