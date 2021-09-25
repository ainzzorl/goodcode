---
layout: post
title:  "New Article: Leader Election with Raft in Scylla"
date: 2021-09-25T18:02:00+0300
---

## [{{ page.title }}]({{ site.baseurl }}{{ page.url }})

<time datetime="{{ page.date | date: "%Y-%m-%d" }}">{{ page.date | date_to_long_string }}</time>

A [new article]({{ site.baseurl }}/articles/scylla-raft-leader-election) about leader election with Raft in [ScyllaDB](https://www.scylladb.com/) has been added to the catalog.

[Raft](https://en.wikipedia.org/wiki/Raft_(algorithm)) is a consensus algorithm that was designed to be easy to understand. In Raft all interactions with clients go through the leader node, which first needs to be elected. In this article, we look into how this is implemented in Scylla.

The code is written in C++.

[Scylla - Leader Election with Raft]({{ site.baseurl }}/articles/scylla-raft-leader-election)

{% include need-your-help.html %}
