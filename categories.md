---
layout: default
title: Categories
permalink: /categories/
---

# Categories

Browse posts by their main subject.

<section id="image-processing" class="category-archive">
  <h2>
    <span class="post-category-badge badge-image-processing">
      Image Processing
    </span>
  </h2>

  <ul class="category-post-list">
    {% for post in site.posts %}
      {% if post.categories contains "image-processing" %}
      <li>
        <a href="{{ post.url | relative_url }}">
          {{ post.title }}
        </a>

        <span class="post-meta">
          {{ post.date | date: site.minima.date_format }}
        </span>
      </li>
      {% endif %}
    {% endfor %}
  </ul>
</section>

<section id="pacs-dicom" class="category-archive">
  <h2>
    <span class="post-category-badge badge-pacs-dicom">
      PACS / DICOM
    </span>
  </h2>

  <ul class="category-post-list">
    {% for post in site.posts %}
      {% if post.categories contains "medical-imaging"
          or post.categories contains "dicom" %}
      <li>
        <a href="{{ post.url | relative_url }}">
          {{ post.title }}
        </a>

        <span class="post-meta">
          {{ post.date | date: site.minima.date_format }}
        </span>
      </li>
      {% endif %}
    {% endfor %}
  </ul>
</section>
