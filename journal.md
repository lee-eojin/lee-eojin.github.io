---
layout: article
titles:
  en: "Journal"
show_title: false
---

### Journal posts

<div class="post-list">
  <ul>
    {%- for post in site.tags["Journal"]-%}
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
