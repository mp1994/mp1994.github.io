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

<script async src="//static.getclicky.com/101356528.js"></script>
<noscript><p><img alt="Clicky" width="1" height="1" src="//in.getclicky.com/101356528ns.gif" /></p></noscript>