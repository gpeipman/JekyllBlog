---
layout: default
title: ASP.NET Core
permalink: /aspnet-core/
---

<h1>ASP.NET Core</h1>

<ul class="posts">
{% for post in site.posts %}
	{% if post.categories contains "ASP.NET Core" %}
		<li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
	{% endif %}
{% endfor %}
</ul>