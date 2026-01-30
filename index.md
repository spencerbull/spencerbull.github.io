---
layout: default
title: Home
---

# Hey, I'm Spencer 👋

Software engineer into AI, robotics, and building things that work.

## Recent Posts

{% for post in site.posts limit:5 %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%B %d, %Y" }}
{% endfor %}

## Links

- [GitHub](https://github.com/spencerbull)
- [LinkedIn](https://linkedin.com/in/spencerbull)
