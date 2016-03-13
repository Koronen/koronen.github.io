---
title: Victor Koronen
subtitle: Web developer, Rubyist, Vimmer, Git addict, GNU/Linux user
layout: default
---
<h3>Archives</h3>
<ul>
  {% for post in site.posts %}
  <li>
    <a href="{{ post.url }}">{{ post.title }}</a>
  </li>
  {% endfor %}
</ul>
