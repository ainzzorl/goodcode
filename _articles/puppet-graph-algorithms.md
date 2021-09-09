---
title:  "Puppet - Graph Algorithms [Ruby]"
layout: default
last_modified_date: 2021-09-09T14:51:00+0300
nav_order: 18

status: PUBLISHED
language: Ruby
short-title: Graph Algorithms
project:
    name: Puppet
    key: puppet
    home-page: https://github.com/puppetlabs/puppet
tags: [graph, algorithm]
---

{% include article-meta.html article=page %}

## Context

*Puppet, an automated administrative engine for your Linux, Unix, and Windows systems, performs administrative tasks (such as adding users, installing packages, and updating server configurations) based on a centralized specification.*

Puppet deals with resources: files, users, services, packages and so on. Resources are defined in manifest files.

*By default, Puppet applies unrelated resources in the order in which they're written in the manifest. If a resource must be applied before or after some other resource, declare a relationship between them to show that their order isn't coincidental. You can also make changes in one resource cause a refresh of some other resource. See the [Relationships and ordering](https://tech-pubs-pdf.s3-us-west-2.amazonaws.com/puppet/puppet-latest/puppet.pdf#%5B%7B%22num%22%3A6183%2C%22gen%22%3A0%7D%2C%7B%22name%22%3A%22XYZ%22%7D%2C56.692%2C261.658%2Cnull%5D) page for more information.*

## Problem

In what order should Puppet apply resources to respect the relationships? How can it detect and report circular dependencies between resources? How does it know everything a resource depends on, directly or indirectly, or everything that depends on it?

## Overview

Puppet represents relationships between resources as a graph with vertices being resources and edges being relationships. Then the problems above can be solved with standard graph algorithms.

In this article we review Puppet's implementation of generic graph algorithms and don't touch the application logic on top of them.

## Implementation details

### Graph definition

Generic graph algorithms are implemented in [`SimpleGraph`](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L395), which is extended by more app-specific [`RelationshipGraph`](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/relationship_graph.rb#L7) and [`Catalog`](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/resource/catalog.rb#L14).

The graph stores:
* `@in_to` - a map from vertices to incoming edges.
* `@out_from` - a map from vertices to outgoing edges.
* `@upstream_from` - a map from vertices to vertices it's reachable from.
* `@downstream_from` - a map from vertices to reachable vertices.

```ruby
  #
  # All public methods of this class must maintain (assume ^ ensure) the following invariants, where "=~=" means
  # equiv. up to order:
  #
  #   @in_to.keys =~= @out_to.keys =~= all vertices
  #   @in_to.values.collect { |x| x.values }.flatten =~= @out_from.values.collect { |x| x.values }.flatten =~= all edges
  #   @in_to[v1][v2] =~= @out_from[v2][v1] =~= all edges from v1 to v2
  #   @in_to   [v].keys =~= vertices with edges leading to   v
  #   @out_from[v].keys =~= vertices with edges leading from v
  #   no operation may shed reference loops (for gc)
  #   recursive operation must scale with the depth of the spanning trees, or better (e.g. no recursion over the set
  #       of all vertices, etc.)
  #
  # This class is intended to be used with DAGs.  However, if the
  # graph has a cycle, it will not cause non-termination of any of the
  # algorithms.
  #
  def initialize
    @in_to = {}
    @out_from = {}
    @upstream_from = {}
    @downstream_from = {}
  end
```

### Adding and removing vertices and edges

[Adding a vertex](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L271-L275):

```ruby
  # Add a new vertex to the graph.
  def add_vertex(vertex)
    @in_to[vertex]    ||= {}
    @out_from[vertex] ||= {}
  end
```

[Removing a vertex](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L277-L285). Besides removing the vertex from the maps, it removes all adjacent edges and also clears reachable vertices - more about this [below](#dependency-closure).

```ruby
  # Remove a vertex from the graph.
  def remove_vertex!(v)
    return unless vertex?(v)
    @upstream_from.clear
    @downstream_from.clear
    (@in_to[v].values+@out_from[v].values).flatten.each { |e| remove_edge!(e) }
    @in_to.delete(v)
    @out_from.delete(v)
  end
```

[Adding an edge](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L297-L311). It adds both adjacent vertices, resets reachable vertices and updates the edge maps. The last 4 lines may look like dead code, but they are not - note the assignments to `@in_to` and `@out_to` in the right-hand sides.

```ruby
  # Add a new edge.  The graph user has to create the edge instance,
  # since they have to specify what kind of edge it is.
  def add_edge(e,*a)
    return add_relationship(e,*a) unless a.empty?
    e = Puppet::Relationship.from_data_hash(e) if e.is_a?(Hash)
    @upstream_from.clear
    @downstream_from.clear
    add_vertex(e.source)
    add_vertex(e.target)
    # Avoid multiple lookups here. This code is performance critical
    arr = (@in_to[e.target][e.source] ||= [])
    arr << e unless arr.include?(e)
    arr = (@out_from[e.source][e.target] ||= [])
    arr << e unless arr.include?(e)
  end
```

[Removing an edge](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L335-L343):

```ruby
  # Remove an edge from our graph.
  def remove_edge!(e)
    if edge?(e.source,e.target)
      @upstream_from.clear
      @downstream_from.clear
      @in_to   [e.target].delete e.source if (@in_to   [e.target][e.source] -= [e]).empty?
      @out_from[e.source].delete e.target if (@out_from[e.source][e.target] -= [e]).empty?
    end
  end
```

### Dependency closure

[Finding](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L41-L44) all dependencies of a resource requires finding all vertices the resource is reachable from.

The comment appears to be wrong - it's the definition of a **dependent**, not a **dependency** - but the code is correct.

```ruby
  # Which resources depend upon the given resource.
  def dependencies(resource)
    vertex?(resource) ? upstream_from_vertex(resource).keys : []
  end
```

Simple recursive algorithm to do it. The results are caches in `@upstream_from`, which is reset whenever an edge is added or removed anywhere in the graph.
```ruby
  def upstream_from_vertex(v)
    return @upstream_from[v] if @upstream_from[v]
    result = @upstream_from[v] = {}
    @in_to[v].keys.each do |node|
      result[node] = 1
      result.update(upstream_from_vertex(node))
    end
    result
  end
```

[Dependents](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L46-L48) are found the same way.

### Tarjan's algorithm and finding cycles

[Implementation](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L94-L155) of [Tarjan's](Tarjan's strongly connected components algorithm) strongly connected components algorithm. The implementation is very close to the description on Wikipedia. The pseudo-code presented on Wikipedia stores values, like `index` or `lowlink`, right on the node. Here it is not possible, so, as noted in the comments, the state is stored in a separate map.

```ruby
  # This is a simple implementation of Tarjan's algorithm to find strongly
  # connected components in the graph; this is a fairly ugly implementation,
  # because I can't just decorate the vertices themselves.
  #
  # This method has an unhealthy relationship with the find_cycles_in_graph
  # method below, which contains the knowledge of how the state object is
  # maintained.
  def tarjan(root, s)
    # initialize the recursion stack we use to work around the nasty lack of a
    # decent Ruby stack.
    recur = [{ :node => root }]

    while not recur.empty? do
      frame = recur.last
      vertex = frame[:node]

      case frame[:step]
      when nil then
        s[:index][vertex]   = s[:number]
        s[:lowlink][vertex] = s[:number]
        s[:number]          = s[:number] + 1

        s[:stack].push(vertex)
        s[:seen][vertex] = true

        frame[:children] = adjacent(vertex)
        frame[:step]     = :children

      when :children then
        if frame[:children].length > 0 then
          child = frame[:children].shift
          if ! s[:index][child] then
            # Never seen, need to recurse.
            frame[:step] = :after_recursion
            frame[:child] = child
            recur.push({ :node => child })
          elsif s[:seen][child] then
            s[:lowlink][vertex] = [s[:lowlink][vertex], s[:index][child]].min
          end
        else
          if s[:lowlink][vertex] == s[:index][vertex] then
            this_scc = []
            loop do
              top = s[:stack].pop
              s[:seen][top] = false
              this_scc << top
              break if top == vertex
            end
            s[:scc] << this_scc
          end
          recur.pop               # done with this node, finally.
        end

      when :after_recursion then
        s[:lowlink][vertex] = [s[:lowlink][vertex], s[:lowlink][frame[:child]]].min
        frame[:step] = :children

      else
        fail "#{frame[:step]} is an unknown step"
      end
    end
  end
```

The strongly connected components are [used](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L157-L189) to find cycles in the graph - any strongly connected component with more than one vertex contains a cycle. Cycles are also possible if a vertex declares a dependency on itself. If there are no circular dependencies, then the graph is a [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph) (Directed Acyclic Graph).

```ruby
  # Find all cycles in the graph by detecting all the strongly connected
  # components, then eliminating everything with a size of one as
  # uninteresting - which it is, because it can't be a cycle. :)
  #
  # This has an unhealthy relationship with the 'tarjan' method above, which
  # it uses to implement the detection of strongly connected components.
  def find_cycles_in_graph
    state = {
      :number => 0, :index => {}, :lowlink => {}, :scc => [],
      :stack => [], :seen => {}
    }

    # we usually have a disconnected graph, must walk all possible roots
    vertices.each do |vertex|
      if ! state[:index][vertex] then
        tarjan vertex, state
      end
    end

    # To provide consistent results to the user, given that a hash is never
    # assured to return the same order, and given our graph processing is
    # based on hash tables, we need to sort the cycles internally, as well as
    # the set of cycles.
    #
    # Given we are in a failure state here, any extra cost is more or less
    # irrelevant compared to the cost of a fix - which is on a human
    # time-scale.
    state[:scc].select do |component|
      multi_vertex_component?(component) || single_vertex_referring_to_self?(component)
    end.map do |component|
      component.sort
    end.sort
  end
```

### Traversing the graph

[Traversing](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L352-L369) a graph from `source`. Passing `direction` allows walking the graph both "up" and "down". The [comment](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/lib/puppet/graph/simple_graph.rb#L354) appears to be wrong - it's a depth-first search ([DFS](https://en.wikipedia.org/wiki/Depth-first_search)), not breadth-first ([BFS](https://en.wikipedia.org/wiki/Breadth-first_search)). There's an easy way to distinguish non-recursive DFS from BFS: DFS uses [stacks](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)) (last-in-first-out) and BFS uses [queues](https://en.wikipedia.org/wiki/Queue_(abstract_data_type)) (first-in-first-out).

```ruby
  # Just walk the tree and pass each edge.
  def walk(source, direction)
    # Use an iterative, breadth-first traversal of the graph. One could do
    # this recursively, but Ruby's slow function calls and even slower
    # recursion make the shorter, recursive algorithm cost-prohibitive.
    stack = [source]
    seen = Set.new
    until stack.empty?
      node = stack.shift
      next if seen.member? node
      connected = adjacent(node, :direction => direction)
      connected.each do |target|
        yield node, target
      end
      stack.concat(connected)
      seen << node
    end
  end
```

## Testing

The [test suite](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/spec/unit/graph/simple_graph_spec.rb) is made up of a large number of very short and very clear tests. E.g. [one of the tests](https://github.com/puppetlabs/puppet/blob/07f3b2baca2b4db23d5464d9d601f649c1213991/spec/unit/graph/simple_graph_spec.rb#L351-L356) for finding cycles:

```ruby
    it "should report multi-vertex loops" do
      add_edges :a => :b, :b => :c, :c => :a
      expect(Puppet).to receive(:err).with(/Found 1 dependency cycle:\n\(Notify\[a\] => Notify\[b\] => Notify\[c\] => Notify\[a\]\)/)
      cycle = @graph.report_cycles_in_graph.first
      expect_cycle_to_include(cycle, :a, :b, :c)
    end
```

## Related

See our article about [graph algorithms in Terraform]({{ site.baseurl }}/articles/terraform-graph-algorithms). [Terraform](https://www.terraform.io/) is another open-source infrastructure management tool, albeit its use cases are quite different than Puppet's. It implements many of the same ideas for similar purposes, but in Go.

## References

* [GitHub Repo](https://github.com/puppetlabs/puppet)
* [Puppet Website](https://puppet.com/)
* [Puppet Documentation](https://tech-pubs-pdf.s3-us-west-2.amazonaws.com/puppet/puppet-latest/puppet.pdf)

## Copyright notice

Puppet is licensed under the [Apache-2.0 License](https://github.com/puppetlabs/puppet/blob/main/LICENSE).

Copyright (c) 2011 Puppet Inc.
