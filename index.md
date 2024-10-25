---
layout: default
---

## Welcome to Laurent Dufresne (Ziox) blog

### Recent Posts

{% for post in site.posts %}
[{{ post.title }}]({{ post.url }})
{% endfor %}
