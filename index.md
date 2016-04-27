---
layout: default
title: "Blog"
---

# Blog:

{% for post in site.posts %}
### *{{ post.date | date: "%b %-d, %Y" }}*: [{{ post.title }}]({{ post.url | prepend: site.baseurl }})

{{ post.excerpt }}
{% endfor %}

