---
layout: article
titles:
  en: "Study"
show_title: false
---

### Study posts

<div class="post-list">
  <ul>
    {%- for post in site.tags["Study"]-%}
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
