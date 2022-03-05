---
layout: home
---

#### _According to Matti_... Tech-related random-access memories.

&nbsp; <!-- vertical space -->

<hr/>
<p>
{% for post in site.posts %}
    <a href="{{ post.url }}">{{ post.title }}</a>
    {{ post.excerpt }}
<hr/>
{% endfor %}
</p>