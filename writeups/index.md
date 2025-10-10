---
layout: page
title: Write-ups
permalink: /writeups/
---

{% assign posts = site.writeups | sort: 'date' | reverse %}
<ul>
{% for p in posts %}
  <li><a href="{{ p.url }}">{{ p.title }}</a> — <small>{{ p.date | date: "%Y-%m-%d" }}</small></li>
{% endfor %}
</ul>
