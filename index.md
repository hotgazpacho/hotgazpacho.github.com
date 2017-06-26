---
layout: default
title: hotgazpacho
---
# Moved to github&hellip;
&hellip;I've moved this blog to github. Some of my posts' formatting are broken,
but I'm working on fixing them. If you find one that is, drop me an email and I'll add it to the queue!

### Blog Posts
<ul>
  {% for post in site.posts %}
    <li{% if page.url == post.url %} class="active"{% endif %}><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>
