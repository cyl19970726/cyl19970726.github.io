---
layout: home
title: HHH Blog
---

# Introduction

Here I share my thoughts on technology, programming, and other interesting topics.

## Recent Posts

{% for post in site.posts limit:5 %}
  <div class="post-preview">
    <h2>
      <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    </h2>
    <span class="post-date">{{ post.date | date: "%B %d, %Y" }}</span>
    {{ post.excerpt }}
    <a href="{{ post.url | relative_url }}">Read More</a>
  </div>
  <hr>
{% endfor %}


## About Me

I'm a technology enthusiast interested in AI, blockchain, and web development. This blog is a place for me to share my projects, ideas, and learning experiences.

[Learn more about me](/about) 