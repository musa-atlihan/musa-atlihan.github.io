---
layout: default
title: Home
permalink: /
---

<section class="hero-card">
  <div class="hero-copy">
    <p class="eyebrow">Personal portfolio and blog</p>
    <h1>Musa Atlıhan</h1>
    <p class="hero-text">
      I write about projects, learning notes, and software development while
      building a personal archive of practical work.
    </p>
    <div class="hero-actions">
      <a class="button primary" href="{{ '/blog/' | relative_url }}">Read the blog</a>
      <a class="button secondary" href="https://github.com/musa-atlihan">GitHub profile</a>
    </div>
  </div>
  <div class="hero-visual" aria-hidden="true">
    <img src="https://github.com/musa-atlihan.png" alt="">
  </div>
</section>

<section class="section-block">
  <div class="section-heading">
    <p class="eyebrow">Latest posts</p>
    <h2>Recent writing</h2>
  </div>

  <div class="post-feed">
    {% for post in site.posts limit:3 %}
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
        <p>New writing will appear here soon.</p>
      </article>
    {% endfor %}
  </div>
</section>
