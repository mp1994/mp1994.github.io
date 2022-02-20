---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
title: Home
---

<html>
  {% for post in site.posts %}
    
      <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
      {{ post.content | truncatewords: 100 }}
    
  {% endfor %}

<hr/>
