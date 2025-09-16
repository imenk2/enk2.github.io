---
layout: default
---

<div class="linked-headings-container">
  <h1><a href="./coding-page.html">Coding</a></h1>
  <h1><a href="./project-page.html">Project</a></h1>
  <h1><a href="./question-page.html">Question</a></h1>
</div>

***

<h3>最新更新的文章</h3>
<ul>
  {% assign pages_in_docs = site.pages | where_exp: "page", "page.path contains 'docs/'" %}
  {% assign updated_pages = pages_in_docs | where_exp: "page", "page.last_modified_at" | sort: "last_modified_at" | reverse %}
  {% assign regular_pages = pages_in_docs | where_exp: "page", "page.last_modified_at == nil" | sort: "date" | reverse %}
  {% assign all_sorted_pages = updated_pages | concat: regular_pages %}

  {% for page in all_sorted_pages limit:3 %}
    <li>
      <a href="{{ page.url | relative_url }}">{{ page.title }}</a>
      - <small>
        {% if page.last_modified_at %}
          更新于: {{ page.last_modified_at | date: "%Y-%m-%d" }}
        {% else %}
          发布于: {{ page.date | date: "%Y-%m-%d" }}
        {% endif %}
      </small>
    </li>
  {% endfor %}
</ul>



***

<script src="https://giscus.app/client.js"
        data-repo="imenk2/enk2"
        data-repo-id="R_kgDOPuZCdQ"
        data-category="Comments"
        data-category-id="DIC_kwDOPuZCdc4CvgCE"
        data-mapping="og:title"
        data-strict="0"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="bottom"
        data-theme="preferred_color_scheme"
        data-lang="zh-CN"
        crossorigin="anonymous"
        async>
</script>
