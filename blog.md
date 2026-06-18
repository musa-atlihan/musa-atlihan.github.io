---
layout: default
title: Blog
permalink: /blog/
---

<section class="section-block">
  <div class="section-heading">
    <p class="eyebrow">Blog</p>
    <h1>All posts</h1>
    <p class="section-intro">
      Notes on projects, learning, and software development.
    </p>
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
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        {% if post.excerpt %}
          <p>{{ post.excerpt | strip_html | truncate: 220 }}</p>
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
        <h2>No posts yet</h2>
        <p>New writing will appear here soon.</p>
      </article>
    {% endfor %}
  </div>
</section>
