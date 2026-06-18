---
layout: default
title: Home
permalink: /
---

<section class="section-block">
  <div class="section-heading">
    <h2>Posts</h2>
  </div>

  <div class="post-feed">
    {% for post in site.posts %}
      <article class="post-card">
        <div class="post-meta">
          <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d, %Y" }}</time>
          {% if post.categories and post.categories.size > 0 %}
            <span>{{ post.categories | join: ", " }}</span>
          {% endif %}
        </div>
        <h3><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h3>
        {% if post.excerpt %}
          <p>{{ post.excerpt | strip_html | truncate: 180 }}</p>
        {% endif %}
        {% if post.tags and post.tags.size > 0 %}
          <div class="tag-list">
            {% for tag in post.tags %}
              <span>#{{ tag }}</span>
            {% endfor %}
          </div>
        {% endif %}
      </article>
    {% else %}
      <article class="post-card">
        <h3>No posts yet</h3>
        <p>New posts will appear here.</p>
      </article>
    {% endfor %}
  </div>
</section>
