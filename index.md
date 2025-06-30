---
layout: default
title: Welcome
---

I'm a passionate software engineer with expertise in distributed systems, including databases and programming language runtimes. I'm committed to continuous learning and actively share my knowledge and ideas with others.

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
      <small>({{ post.date | date: "%Y-%m-%d" }})</small>
    </li>
  {% endfor %}
</ul>
