---
layout: archive
permalink: /hive/
title: "Latest Posts in *hive*"
excerpt: "hive、impala的基础和优化"
---

<div class="tiles">
{% for post in site.posts %}
	{% if post.categories contains 'hive' %}
		{% include post-grid.html %}
	{% endif %}
{% endfor %}
</div><!-- /.tiles -->
