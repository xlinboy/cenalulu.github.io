---
layout: archive
permalink: /hadoop/
title: "Latest Posts in *hadoop*"
excerpt: "I make living from IT"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'hadoop' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->

