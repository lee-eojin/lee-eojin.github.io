---
layout: article
titles:
  en: "Wooteco"
show_title: false
---

### Wooteco posts

<div class="post-list">
  <ul>
    {%- for post in site.tags["Wooteco"]-%}
    <li>
      {%- assign __path = post.url -%}
      {%- include snippets/prepend-baseurl.html -%}
      {%- assign href = __return -%}
      <div>
        <h4><a href="{{ href }}">{{ post.title }}</a></h4>
      </div>
    </li>
    {%- endfor -%}
  </ul>
</div>

<script>
  {%- include scripts/home.js -%}
</script>
