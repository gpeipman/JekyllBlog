---
layout: default
title: IoT
permalink: /iot/
---

<h1>IoT</h1>

<ul class="posts">
{% for post in site.posts %}
	{% if post.categories contains "IoT" %}
		<li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
	{% endif %}
{% endfor %}
</ul>