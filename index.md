---
layout: default
title: Posts
---

<section class="intro">
  <h1>AI Systems Research Notes</h1>
  <p>Public notes on memory, state, architecture, and deployment systems.</p>
</section>

<h2>All Posts</h2>

{% if site.posts.size > 0 %}
  <ul class="post-list">
    {% for post in site.posts %}
      <li>
        <a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
        <span class="post-list-meta">
          <span class="lang-badge">{{ post.lang | default: "en" | upcase }}</span>
          <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
        </span>
      </li>
    {% endfor %}
  </ul>
{% else %}
  <p>No public posts yet.</p>
{% endif %}
