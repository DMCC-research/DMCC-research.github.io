---
layout: default
title: 领域
permalink: /zh/domains/
lang: zh
translation_url: /domains/
---

# 研究领域

<div class="domain-grid">
  {% for domain in site.data.domains %}
    <a class="domain-link" href="{{ domain.path_zh | relative_url }}">
      <span>{{ domain.id }}</span>
      {{ domain.title_zh | escape }}
    </a>
  {% endfor %}
</div>
