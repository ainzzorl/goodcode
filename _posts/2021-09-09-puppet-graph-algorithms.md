---
layout: post
title:  "New Article: Graph Algorithms in Puppet"
date: 2021-09-09T14:53:00+0300
---

## [{{ page.title }}]({{ site.baseurl }}{{ page.url }})

<time datetime="{{ page.date | date: "%Y-%m-%d" }}">{{ page.date | date_to_long_string }}</time>

A [new article]({{ site.baseurl }}/articles/puppet-graph-algorithms) about graph algorithms in [Puppet](https://puppet.com) has been added to the catalog. This is our second article featuring Puppet after [Puppet - HTTP Connection Pool]({{ site.baseurl }}/articles/puppet-connection-pool).

We discuss how Puppet uses standard graph algorithms, such as [Tarjan's](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm) strongly connected components algorithm, to manage dependencies between resources. It strongly resembles our older article about [graph algorithms in Terraform]({{ site.baseurl }}/articles/terraform-graph-algorithms) (if you read this article long ago, consider giving it another look - we recently made significant updates to it). We consider continuing reviewing uses of graph algorithms in popular open-source projects.

The code is written in Ruby.

[Puppet - Graph Algorithms]({{ site.baseurl }}/articles/puppet-graph-algorithms)

{% include need-your-help.html %}
