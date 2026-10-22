---
layout: default
title: topics
permalink: /tags/
---
<header class="post-head">
  <h1 class="post-title">Topics</h1>
  <p class="post-meta">every tag used across writeups</p>
</header>

<div class="tag-cloud">
  {% assign tags = site.tags | sort %}
  {% for tag in tags %}
    <span id="{{ tag[0] | slugify }}"></span>
    <a class="tag-pill" href="#{{ tag[0] | slugify }}">{{ tag[0] }} ({{ tag[1] | size }})</a>
  {% endfor %}
</div>

{% assign tags = site.tags | sort %}
{% for tag in tags %}
  <div class="index__year">
    <span class="index__year-num">{{ tag[0] }}</span>
    <span class="index__year-rule"></span>
  </div>
  <ul class="index__rows">
    {% for post in tag[1] %}
      <li class="row">
        <time class="row__date">{{ post.date | date: "%Y.%m.%d" }}</time>
        <span class="row__bar" aria-hidden="true"></span>
        <div class="row__content">
          <a class="row__title" href="{{ post.url | relative_url }}">{{ post.title }}</a>
        </div>
      </li>
    {% endfor %}
  </ul>
{% endfor %}
