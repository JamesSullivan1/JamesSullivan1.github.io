---
layout: page
title: James Sullivan's Blog
tagline: Supporting tagline
---
{% include JB/setup %}

<ul class="posts">
{% for post in site.posts %}
<li>
    <div class="post-preview">
        <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
        <div class="post-date">{{ post.date | date: "%B %d, %Y" }}</div>
        {{ post.content | split:'<!--break-->' | first }}
        {% if post.content contains '<!--break-->' %}
            <a href="{{ post.url }}">read more</a>
        {% endif %}
    </div>
    <hr>
</li>
{% endfor %}
</ul>

