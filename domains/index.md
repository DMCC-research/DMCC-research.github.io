---
layout: default
title: Domains
permalink: /domains/
lang: en
translation_url: /zh/domains/
---

# Research Domains

<div class="domain-grid">
  {% for domain in site.data.domains %}
    <a class="domain-link" href="{{ domain.path | relative_url }}">
      <span>{{ domain.id }}</span>
      {{ domain.title | escape }}
    </a>
  {% endfor %}
</div>
