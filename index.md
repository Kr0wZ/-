---
layout: default
title: KrowZ - Tac Au T'Hack
---

### Liste des articles sur le blog :

{% assign sortedCategories = site.categories | sort %}
{% for category in sortedCategories %}
 {% assign cat4url = category[0] | remove:' ' | downcase %}
 <h4>{{category[0]}}</h4>
 
<ul>
  {% for post in site.posts %}
    {% if post.categories[0] == category[0] %}
  <li>
    <a href="/fr{{ post.url }}">{{ post.title }}</a>
  </li>
      {% endif %}
  {% endfor %}
</ul>

{% endfor %}
