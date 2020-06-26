---
layout: default
---

### Liste des articles sur le blog :

<ul>
  {% for post in site.posts %}
    <li>
      <a href="/fr{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
