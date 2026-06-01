---
layout: default
title: Posts
---

<section class="intro">
  <h1>Data Movement-Centric Computing</h1>
  <p>Public research notes on memory, state, and data movement in modern AI deployment systems.</p>
</section>

<h2>All Posts</h2>

{% if site.posts.size > 0 %}
  <ul class="post-list">
    {% for post in site.posts %}
      <li>
        <a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
        <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
      </li>
    {% endfor %}
  </ul>
{% else %}
  <p>No public posts yet.</p>
{% endif %}
