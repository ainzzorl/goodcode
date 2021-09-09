---
layout: post
title:  "New Article: Decorating Artifact Caches in Buck"
date: 2021-08-13T12:19:00+0300
---

<article>
  <h2>
    <a href="{{ site.baseurl }}{{ page.url }}">
      {{ page.title }}
    </a>
  </h2>
  <time datetime="{{ page.date | date: "%Y-%m-%d" }}">{{ page.date | date_to_long_string }}</time>

  <p>
    A <a href="{{ site.baseurl }}/articles/buck-artifact-cache-decorators">new article</a> about decorators for artifact caches in <a href="https://buck.build/">Buck</a>, a multi-language build system developed and used by Facebook, has been added to the catalog.
  </p>

  <p>
      The code is written in Java. It uses the classical <a href="https://en.wikipedia.org/wiki/Decorator_pattern">Decorator</a> design pattern to add behavior to individual cache instances without affecting the behavior of other objects from the same class.
  </p>

  <p>
    <a href="{{ site.baseurl }}/articles/buck-artifact-cache-decorators">Buck - Artifact Cache Decorators</a>
  </p>
</article>

{% include need-your-help.html %}

