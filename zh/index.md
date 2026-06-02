---
layout: default
title: 文章
permalink: /zh/
lang: zh
translation_url: /
---

<section class="intro">
  <h1>AI 系统研究笔记</h1>
  <p>关于记忆、状态、架构和部署系统的公开研究笔记。</p>
</section>

<h2>全部文章</h2>

{% assign posts = site.posts | where: "lang", "zh" %}
{% if posts.size > 0 %}
  <ul class="post-list">
    {% for post in posts %}
      <li>
        <a href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
        <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%Y-%m-%d" }}</time>
      </li>
    {% endfor %}
  </ul>
{% else %}
  <p>暂无公开文章。</p>
{% endif %}
