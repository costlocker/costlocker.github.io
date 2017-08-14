---
title: Costlocker
perex: API documentation, blog, sample apps, ...
links:
  API Reference: http://docs.costlocker.apiary.io/
  Blog: "/blog/"
  Costlocker.com: https://costlocker.com
  Costlocker.cz: http://costlocker.cz
  Github: "https://github.com/costlocker"
---

# Costlocker

This is the official repository for **Costlocker API**.

## API documentation

<div class="app">
  <a href="http://docs.costlocker.apiary.io/#" target="_blank">
      <figure>
          <img src="https://static.apiary.io/assets/v6Zkz37_.png" />
      </figure>
      <span>Costlocker API Reference (apiary)</span>
  </a>
</div>

## Blog

<div id="blog">
  {% for post in site.posts %}
  <div class="app">
    <a href="{{ post.url }}">
        <figure>
            <img src="{{ post.icon }}" />
        </figure>
        <span>{{ post.title }}</span>
    </a>
  </div>
  {% endfor %}
</div>

## Sample apps

{% for group in site.data.apps %}

#### {{ group.name }}

{% for app in group.apps %}
<div class="app">
  <a href="{{ app.web }}" target="_blank">
    <figure>
      <img src="{{ app.icon }}" title="{{ app.name }}" />
    </figure>
    <span>{{ app.description }}</span>
  </a>
  <a href="{{ app.code }}" class="code">[source code]</a>
</div>
{% endfor %}

{% endfor %}

## Contributions

Help us make the API better! Use Github [issues](https://github.com/costlocker/costlocker.github.io/issues)
for reporting bugs and typos. Also, let us know about your integration and send a 
[pull request](https://github.com/costlocker/costlocker.github.io/pulls). Thanks!
