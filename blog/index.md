---
title: Development Blog
perex: Articles about integrating Costlocker API
links:
  Developers portal: "/"
  API Reference: http://docs.costlocker.apiary.io/
  EN blog: https://costlocker.com/blog/
  CZ blog: http://costlocker.cz/blog/
---

{% for post in site.posts %}
<h2><a href="{{ post.url }}">{{ post.title }}</a></h2>

<blockquote>
{{ post.perex }}<br />
<small>{{ post.date | date: '%B %-d, %Y' }}</small>
</blockquote>
<img src="{{ post.image }}" alt="" />
{% endfor %}
