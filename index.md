---
# You don't need to edit this file, it's empty on purpose.
# Edit theme's home layout instead if you wanna make some changes
# See: https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: home
---

{% for post in site.posts %}
### *{{ post.date | date: "%b %-d, %Y" }}*: [{{ post.title }}]({{ post.url | prepend: site.baseurl }})

{{ post.excerpt }}

*[Read more...]({{ post.url | prepend: site.baseurl }})*

<br/>

{% endfor %}

