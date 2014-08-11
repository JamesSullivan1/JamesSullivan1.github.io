---
layout: page
title: "Blog"
description: "My blog"
---
{% include JB/setup %}

<ul class="posts">
{% for post in site.posts %}
<li>
<a href="{{ post.url }}">
<h3>{{ post.title }}</h3>
<p class="blogdate">{{ post.date | date: "%d %B %Y" }}</p>
<div>{{ post.content |truncatehtml | truncatewords: 60 }}</div>
</a>
</li>
{% endfor %}
</ul>

