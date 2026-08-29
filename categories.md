---
layout: default
title: Categories
permalink: /categories/
---

# Categories

Browse posts by their main subject.

{% assign image_processing_count =
  site.categories["image-processing"] | size %}
{% assign pacs_dicom_count =
  site.categories["medical-imaging"] | size %}

<section id="image-processing" class="category-archive">
  <h2 class="category-archive-heading">
    <span class="post-category-badge badge-image-processing">
      Image Processing
    </span>

    <span class="category-post-count">
      {{ image_processing_count }} posts
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
  <h2 class="category-archive-heading">
    <span class="post-category-badge badge-pacs-dicom">
      PACS / DICOM
    </span>

    <span class="category-post-count">
      {{ pacs_dicom_count }} posts
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
