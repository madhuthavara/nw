---
layout: default
title: Home
---

# Posts

{% for post in site.posts %}
- [{{ post.title }}]({{ site.baseurl }}{{ post.url }})  
  <small>{{ post.date | date: "%Y-%m-%d" }}</small>
{% endfor %}
