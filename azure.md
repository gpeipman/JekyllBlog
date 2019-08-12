---
layout: default
title: Azure
permalink: /azure/
---

<h1>Azure</h1>

<ul class="posts">
{% for post in site.posts %}
	{% if post.categories contains "Azure" %}
		<li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
	{% endif %}
{% endfor %}
</ul>