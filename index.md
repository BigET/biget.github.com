---
layout: page
title: A page to put stuff so they are not forgotten.
tagline: Supporting tagline
---
{% include JB/setup %}

## Post list

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

