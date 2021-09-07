---
title:  "Terraform - Graph Algorithms [Go]"
layout: default
last_modified_date: 2021-09-07T20:06:00+0300
nav_order: 3

status: PUBLISHED
language: Go
short-title: Graph Algorithms
project:
    name: Terraform
    key: terraform
    home-page: https://github.com/hashicorp/terraform
tags: [graph, algorithm, infrastructure-as-code]
---

{% include article-meta.html article=page %}

## Context

*Terraform is an open-source [infrastructure as code](https://en.wikipedia.org/wiki/Infrastructure_as_code) software tool that provides a consistent CLI workflow to manage hundreds of cloud services. Terraform codifies cloud APIs into declarative configuration files.*

Terraform deals with [resources](https://www.terraform.io/docs/language/resources/index.html): virtual machine instances, disks, load balancers, etc. Resources can depend on other resources.

## Problem

Given an infrastructure definition, how do we make sure there are no cyclical dependencies between resources? If there are cyclical dependencies, how do we find them and report back to users? In what order should we create resources to guarantee that each resource is created after its dependencies? Which resources can be created in parallel?

## Overview

Terraform represents infrastructure [as a graph](https://www.terraform.io/docs/internals/graph.html) with resources as vertices and dependencies as edges. Then the problems above can be solved with standard graph algorithms.

The requirement that there are no circular dependencies means that the graph is a [DAG](https://en.wikipedia.org/wiki/Directed_acyclic_graph) (Directed Acyclic Graph).

Terraform implements a number of graph algorithms:
* [Depth-first search (DFS)](https://en.wikipedia.org/wiki/Depth-first_search) - a building block for other algorithms.
* [Tarjan's strongly connected components algorithm](https://en.wikipedia.org/wiki/Tarjan%27s_strongly_connected_components_algorithm) - to find cycles in the graph.
* [Transitive reduction](https://en.wikipedia.org/wiki/Transitive_reduction) - to simplify the graph by removing redundant edges.
* Parallel graph walk - to parallelize operations.

## Implementation details

### [Graph structure](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/graph.go#L9-L15)

A graph is represented as a set of edges, a set of vertices, a map of outgoing edges (`downEdges`) and a map of incoming edges (`upEdges`). As Go doesn't have the [Set](https://en.wikipedia.org/wiki/Set_(abstract_data_type)) data structure in its standard library, Terraform implements its own [`Set`](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/set.go) class on top of golang's maps.

```go
// Graph is used to represent a dependency graph.
type Graph struct {
	vertices  Set
	edges     Set
	downEdges map[interface{}]Set
	upEdges   map[interface{}]Set
}
```

### [Connecting](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/graph.go#L196-L231) two vertices


```go
// Connect adds an edge with the given source and target. This is safe to
// call multiple times with the same value. Note that the same value is
// verified through pointer equality of the vertices, not through the
// value of the edge itself.
func (g *Graph) Connect(edge Edge) {
	g.init()

	source := edge.Source()
	target := edge.Target()
	sourceCode := hashcode(source)
	targetCode := hashcode(target)

	// Do we have this already? If so, don't add it again.
	if s, ok := g.downEdges[sourceCode]; ok && s.Include(target) {
		return
	}

	// Add the edge to the set
	g.edges.Add(edge)

	// Add the down edge
	s, ok := g.downEdges[sourceCode]
	if !ok {
		s = make(Set)
		g.downEdges[sourceCode] = s
	}
	s.Add(target)

	// Add the up edge
	s, ok = g.upEdges[targetCode]
	if !ok {
		s = make(Set)
		g.upEdges[targetCode] = s
	}
	s.Add(source)
}
```

### [Finding the root](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L61-L82)

The root in a DAG is the vertex with no incoming edges.

```go
// Root returns the root of the DAG, or an error.
//
// Complexity: O(V)
func (g *AcyclicGraph) Root() (Vertex, error) {
	roots := make([]Vertex, 0, 1)
	for _, v := range g.Vertices() {
		if g.upEdgesNoCopy(v).Len() == 0 {
			roots = append(roots, v)
		}
	}

	if len(roots) > 1 {
		// TODO(mitchellh): make this error message a lot better
		return nil, fmt.Errorf("multiple roots: %#v", roots)
	}

	if len(roots) == 0 {
		return nil, fmt.Errorf("no roots found")
	}

	return roots[0], nil
}
```

### [Transitive reduction](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L84-L114)

```go
// TransitiveReduction performs the transitive reduction of graph g in place.
// The transitive reduction of a graph is a graph with as few edges as
// possible with the same reachability as the original graph. This means
// that if there are three nodes A => B => C, and A connects to both
// B and C, and B connects to C, then the transitive reduction is the
// same graph with only a single edge between A and B, and a single edge
// between B and C.
//
// The graph must be valid for this operation to behave properly. If
// Validate() returns an error, the behavior is undefined and the results
// will likely be unexpected.
//
// Complexity: O(V(V+E)), or asymptotically O(VE)
func (g *AcyclicGraph) TransitiveReduction() {
	// For each vertex u in graph g, do a DFS starting from each vertex
	// v such that the edge (u,v) exists (v is a direct descendant of u).
	//
	// For each v-prime reachable from v, remove the edge (u, v-prime).
	for _, u := range g.Vertices() {
		uTargets := g.downEdgesNoCopy(u)

		g.DepthFirstWalk(g.downEdgesNoCopy(u), func(v Vertex, d int) error {
			shared := uTargets.Intersection(g.downEdgesNoCopy(v))
			for _, vPrime := range shared {
				g.RemoveEdge(BasicEdge(u, vPrime))
			}

			return nil
		})
	}
}
```

### [DFS](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L182-L219)

Unlike many other DFS implementations, Terraform's implementation is not recursive, which should make it more efficient. Note that `frontier` is a [stack](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)).

```go
// DepthFirstWalk does a depth-first walk of the graph starting from
// the vertices in start.
func (g *AcyclicGraph) DepthFirstWalk(start Set, f DepthWalkFunc) error {
	seen := make(map[Vertex]struct{})
	frontier := make([]*vertexAtDepth, 0, len(start))
	for _, v := range start {
		frontier = append(frontier, &vertexAtDepth{
			Vertex: v,
			Depth:  0,
		})
	}
	for len(frontier) > 0 {
		// Pop the current vertex
		n := len(frontier)
		current := frontier[n-1]
		frontier = frontier[:n-1]

		// Check if we've seen this already and return...
		if _, ok := seen[current.Vertex]; ok {
			continue
		}
		seen[current.Vertex] = struct{}{}

		// Visit the current node
		if err := f(current.Vertex, current.Depth); err != nil {
			return err
		}

		for _, v := range g.downEdgesNoCopy(current.Vertex) {
			frontier = append(frontier, &vertexAtDepth{
				Vertex: v,
				Depth:  current.Depth + 1,
			})
		}
	}

	return nil
}
```

There's a very similar method, [`ReverseDepthFirstWalk`](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L264-L301), to walk the graph "up". It is used to [find all descendants](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag.go#L45-L59) of a vertex.

### [Tarjan's algorithm](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/tarjan.go)

Some stack operations are omitted. If the graph is indeed a DAG, each strongly connected component should contain only one vertex.

```go
// StronglyConnected returns the list of strongly connected components
// within the Graph g. This information is primarily used by this package
// for cycle detection, but strongly connected components have widespread
// use.
func StronglyConnected(g *Graph) [][]Vertex {
	vs := g.Vertices()
	acct := sccAcct{
		NextIndex:   1,
		VertexIndex: make(map[Vertex]int, len(vs)),
	}
	for _, v := range vs {
		// Recurse on any non-visited nodes
		if acct.VertexIndex[v] == 0 {
			stronglyConnected(&acct, g, v)
		}
	}
	return acct.SCC
}

func stronglyConnected(acct *sccAcct, g *Graph, v Vertex) int {
	// Initial vertex visit
	index := acct.visit(v)
	minIdx := index

	for _, raw := range g.downEdgesNoCopy(v) {
		target := raw.(Vertex)
		targetIdx := acct.VertexIndex[target]

		// Recurse on successor if not yet visited
		if targetIdx == 0 {
			minIdx = min(minIdx, stronglyConnected(acct, g, target))
		} else if acct.inStack(target) {
			// Check if the vertex is in the stack
			minIdx = min(minIdx, targetIdx)
		}
	}

	// Pop the strongly connected components off the stack if
	// this is a root vertex
	if index == minIdx {
		var scc []Vertex
		for {
			v2 := acct.pop()
			scc = append(scc, v2)
			if v2 == v {
				break
			}
		}

		acct.SCC = append(acct.SCC, scc)
	}

	return minIdx
}

func min(a, b int) int {
	if a <= b {
		return a
	}
	return b
}

// sccAcct is used ot pass around accounting information for
// the StronglyConnectedComponents algorithm
type sccAcct struct {
	NextIndex   int
	VertexIndex map[Vertex]int
	Stack       []Vertex
	SCC         [][]Vertex
}

// visit assigns an index and pushes a vertex onto the stack
func (s *sccAcct) visit(v Vertex) int {
	idx := s.NextIndex
	s.VertexIndex[v] = idx
	s.NextIndex++
	s.push(v)
	return idx
}

// ...
```


{::options parse_block_html="true" /}

### [Parallel walk](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/walk.go#L12-L448)

<details><summary markdown="span">*Expand (very long block of code)*</summary>
{% raw %}
```go
// Walker is used to walk every vertex of a graph in parallel.
//
// A vertex will only be walked when the dependencies of that vertex have
// been walked. If two vertices can be walked at the same time, they will be.
//
// Update can be called to update the graph. This can be called even during
// a walk, changing vertices/edges mid-walk. This should be done carefully.
// If a vertex is removed but has already been executed, the result of that
// execution (any error) is still returned by Wait. Changing or re-adding
// a vertex that has already executed has no effect. Changing edges of
// a vertex that has already executed has no effect.
//
// Non-parallelism can be enforced by introducing a lock in your callback
// function. However, the goroutine overhead of a walk will remain.
// Walker will create V*2 goroutines (one for each vertex, and dependency
// waiter for each vertex). In general this should be of no concern unless
// there are a huge number of vertices.
//
// The walk is depth first by default. This can be changed with the Reverse
// option.
//
// A single walker is only valid for one graph walk. After the walk is complete
// you must construct a new walker to walk again. State for the walk is never
// deleted in case vertices or edges are changed.
type Walker struct {
	// Callback is what is called for each vertex
	Callback WalkFunc

	// Reverse, if true, causes the source of an edge to depend on a target.
	// When false (default), the target depends on the source.
	Reverse bool

	// changeLock must be held to modify any of the fields below. Only Update
	// should modify these fields. Modifying them outside of Update can cause
	// serious problems.
	changeLock sync.Mutex
	vertices   Set
	edges      Set
	vertexMap  map[Vertex]*walkerVertex

	// wait is done when all vertices have executed. It may become "undone"
	// if new vertices are added.
	wait sync.WaitGroup

	// diagsMap contains the diagnostics recorded so far for execution,
	// and upstreamFailed contains all the vertices whose problems were
	// caused by upstream failures, and thus whose diagnostics should be
	// excluded from the final set.
	//
	// Readers and writers of either map must hold diagsLock.
	diagsMap       map[Vertex]tfdiags.Diagnostics
	upstreamFailed map[Vertex]struct{}
	diagsLock      sync.Mutex
}

func (w *Walker) init() {
	if w.vertices == nil {
		w.vertices = make(Set)
	}
	if w.edges == nil {
		w.edges = make(Set)
	}
}

type walkerVertex struct {
	// These should only be set once on initialization and never written again.
	// They are not protected by a lock since they don't need to be since
	// they are write-once.

	// DoneCh is closed when this vertex has completed execution, regardless
	// of success.
	//
	// CancelCh is closed when the vertex should cancel execution. If execution
	// is already complete (DoneCh is closed), this has no effect. Otherwise,
	// execution is cancelled as quickly as possible.
	DoneCh   chan struct{}
	CancelCh chan struct{}

	// Dependency information. Any changes to any of these fields requires
	// holding DepsLock.
	//
	// DepsCh is sent a single value that denotes whether the upstream deps
	// were successful (no errors). Any value sent means that the upstream
	// dependencies are complete. No other values will ever be sent again.
	//
	// DepsUpdateCh is closed when there is a new DepsCh set.
	DepsCh       chan bool
	DepsUpdateCh chan struct{}
	DepsLock     sync.Mutex

	// Below is not safe to read/write in parallel. This behavior is
	// enforced by changes only happening in Update. Nothing else should
	// ever modify these.
	deps         map[Vertex]chan struct{}
	depsCancelCh chan struct{}
}

// Wait waits for the completion of the walk and returns diagnostics describing
// any problems that arose. Update should be called to populate the walk with
// vertices and edges prior to calling this.
//
// Wait will return as soon as all currently known vertices are complete.
// If you plan on calling Update with more vertices in the future, you
// should not call Wait until after this is done.
func (w *Walker) Wait() tfdiags.Diagnostics {
	// Wait for completion
	w.wait.Wait()

	var diags tfdiags.Diagnostics
	w.diagsLock.Lock()
	for v, vDiags := range w.diagsMap {
		if _, upstream := w.upstreamFailed[v]; upstream {
			// Ignore diagnostics for nodes that had failed upstreams, since
			// the downstream diagnostics are likely to be redundant.
			continue
		}
		diags = diags.Append(vDiags)
	}
	w.diagsLock.Unlock()

	return diags
}

// Update updates the currently executing walk with the given graph.
// This will perform a diff of the vertices and edges and update the walker.
// Already completed vertices remain completed (including any errors during
// their execution).
//
// This returns immediately once the walker is updated; it does not wait
// for completion of the walk.
//
// Multiple Updates can be called in parallel. Update can be called at any
// time during a walk.
func (w *Walker) Update(g *AcyclicGraph) {
	w.init()
	v := make(Set)
	e := make(Set)
	if g != nil {
		v, e = g.vertices, g.edges
	}

	// Grab the change lock so no more updates happen but also so that
	// no new vertices are executed during this time since we may be
	// removing them.
	w.changeLock.Lock()
	defer w.changeLock.Unlock()

	// Initialize fields
	if w.vertexMap == nil {
		w.vertexMap = make(map[Vertex]*walkerVertex)
	}

	// Calculate all our sets
	newEdges := e.Difference(w.edges)
	oldEdges := w.edges.Difference(e)
	newVerts := v.Difference(w.vertices)
	oldVerts := w.vertices.Difference(v)

	// Add the new vertices
	for _, raw := range newVerts {
		v := raw.(Vertex)

		// Add to the waitgroup so our walk is not done until everything finishes
		w.wait.Add(1)

		// Add to our own set so we know about it already
		w.vertices.Add(raw)

		// Initialize the vertex info
		info := &walkerVertex{
			DoneCh:   make(chan struct{}),
			CancelCh: make(chan struct{}),
			deps:     make(map[Vertex]chan struct{}),
		}

		// Add it to the map and kick off the walk
		w.vertexMap[v] = info
	}

	// Remove the old vertices
	for _, raw := range oldVerts {
		v := raw.(Vertex)

		// Get the vertex info so we can cancel it
		info, ok := w.vertexMap[v]
		if !ok {
			// This vertex for some reason was never in our map. This
			// shouldn't be possible.
			continue
		}

		// Cancel the vertex
		close(info.CancelCh)

		// Delete it out of the map
		delete(w.vertexMap, v)
		w.vertices.Delete(raw)
	}

	// Add the new edges
	changedDeps := make(Set)
	for _, raw := range newEdges {
		edge := raw.(Edge)
		waiter, dep := w.edgeParts(edge)

		// Get the info for the waiter
		waiterInfo, ok := w.vertexMap[waiter]
		if !ok {
			// Vertex doesn't exist... shouldn't be possible but ignore.
			continue
		}

		// Get the info for the dep
		depInfo, ok := w.vertexMap[dep]
		if !ok {
			// Vertex doesn't exist... shouldn't be possible but ignore.
			continue
		}

		// Add the dependency to our waiter
		waiterInfo.deps[dep] = depInfo.DoneCh

		// Record that the deps changed for this waiter
		changedDeps.Add(waiter)
		w.edges.Add(raw)
	}

	// Process removed edges
	for _, raw := range oldEdges {
		edge := raw.(Edge)
		waiter, dep := w.edgeParts(edge)

		// Get the info for the waiter
		waiterInfo, ok := w.vertexMap[waiter]
		if !ok {
			// Vertex doesn't exist... shouldn't be possible but ignore.
			continue
		}

		// Delete the dependency from the waiter
		delete(waiterInfo.deps, dep)

		// Record that the deps changed for this waiter
		changedDeps.Add(waiter)
		w.edges.Delete(raw)
	}

	// For each vertex with changed dependencies, we need to kick off
	// a new waiter and notify the vertex of the changes.
	for _, raw := range changedDeps {
		v := raw.(Vertex)
		info, ok := w.vertexMap[v]
		if !ok {
			// Vertex doesn't exist... shouldn't be possible but ignore.
			continue
		}

		// Create a new done channel
		doneCh := make(chan bool, 1)

		// Create the channel we close for cancellation
		cancelCh := make(chan struct{})

		// Build a new deps copy
		deps := make(map[Vertex]<-chan struct{})
		for k, v := range info.deps {
			deps[k] = v
		}

		// Update the update channel
		info.DepsLock.Lock()
		if info.DepsUpdateCh != nil {
			close(info.DepsUpdateCh)
		}
		info.DepsCh = doneCh
		info.DepsUpdateCh = make(chan struct{})
		info.DepsLock.Unlock()

		// Cancel the older waiter
		if info.depsCancelCh != nil {
			close(info.depsCancelCh)
		}
		info.depsCancelCh = cancelCh

		// Start the waiter
		go w.waitDeps(v, deps, doneCh, cancelCh)
	}

	// Start all the new vertices. We do this at the end so that all
	// the edge waiters and changes are set up above.
	for _, raw := range newVerts {
		v := raw.(Vertex)
		go w.walkVertex(v, w.vertexMap[v])
	}
}

// edgeParts returns the waiter and the dependency, in that order.
// The waiter is waiting on the dependency.
func (w *Walker) edgeParts(e Edge) (Vertex, Vertex) {
	if w.Reverse {
		return e.Source(), e.Target()
	}

	return e.Target(), e.Source()
}

// walkVertex walks a single vertex, waiting for any dependencies before
// executing the callback.
func (w *Walker) walkVertex(v Vertex, info *walkerVertex) {
	// When we're done executing, lower the waitgroup count
	defer w.wait.Done()

	// When we're done, always close our done channel
	defer close(info.DoneCh)

	// Wait for our dependencies. We create a [closed] deps channel so
	// that we can immediately fall through to load our actual DepsCh.
	var depsSuccess bool
	var depsUpdateCh chan struct{}
	depsCh := make(chan bool, 1)
	depsCh <- true
	close(depsCh)
	for {
		select {
		case <-info.CancelCh:
			// Cancel
			return

		case depsSuccess = <-depsCh:
			// Deps complete! Mark as nil to trigger completion handling.
			depsCh = nil

		case <-depsUpdateCh:
			// New deps, reloop
		}

		// Check if we have updated dependencies. This can happen if the
		// dependencies were satisfied exactly prior to an Update occurring.
		// In that case, we'd like to take into account new dependencies
		// if possible.
		info.DepsLock.Lock()
		if info.DepsCh != nil {
			depsCh = info.DepsCh
			info.DepsCh = nil
		}
		if info.DepsUpdateCh != nil {
			depsUpdateCh = info.DepsUpdateCh
		}
		info.DepsLock.Unlock()

		// If we still have no deps channel set, then we're done!
		if depsCh == nil {
			break
		}
	}

	// If we passed dependencies, we just want to check once more that
	// we're not cancelled, since this can happen just as dependencies pass.
	select {
	case <-info.CancelCh:
		// Cancelled during an update while dependencies completed.
		return
	default:
	}

	// Run our callback or note that our upstream failed
	var diags tfdiags.Diagnostics
	var upstreamFailed bool
	if depsSuccess {
		diags = w.Callback(v)
	} else {
		log.Printf("[TRACE] dag/walk: upstream of %q errored, so skipping", VertexName(v))
		// This won't be displayed to the user because we'll set upstreamFailed,
		// but we need to ensure there's at least one error in here so that
		// the failures will cascade downstream.
		diags = diags.Append(errors.New("upstream dependencies failed"))
		upstreamFailed = true
	}

	// Record the result (we must do this after execution because we mustn't
	// hold diagsLock while visiting a vertex.)
	w.diagsLock.Lock()
	if w.diagsMap == nil {
		w.diagsMap = make(map[Vertex]tfdiags.Diagnostics)
	}
	w.diagsMap[v] = diags
	if w.upstreamFailed == nil {
		w.upstreamFailed = make(map[Vertex]struct{})
	}
	if upstreamFailed {
		w.upstreamFailed[v] = struct{}{}
	}
	w.diagsLock.Unlock()
}

func (w *Walker) waitDeps(
	v Vertex,
	deps map[Vertex]<-chan struct{},
	doneCh chan<- bool,
	cancelCh <-chan struct{}) {

	// For each dependency given to us, wait for it to complete
	for dep, depCh := range deps {
	DepSatisfied:
		for {
			select {
			case <-depCh:
				// Dependency satisfied!
				break DepSatisfied

			case <-cancelCh:
				// Wait cancelled. Note that we didn't satisfy dependencies
				// so that anything waiting on us also doesn't run.
				doneCh <- false
				return

			case <-time.After(time.Second * 5):
				log.Printf("[TRACE] dag/walk: vertex %q is waiting for %q",
					VertexName(v), VertexName(dep))
			}
		}
	}

	// Dependencies satisfied! We need to check if any errored
	w.diagsLock.Lock()
	defer w.diagsLock.Unlock()
	for dep := range deps {
		if w.diagsMap[dep].HasErrors() {
			// One of our dependencies failed, so return false
			doneCh <- false
			return
		}
	}

	// All dependencies satisfied and successful
	doneCh <- true
}
```
{% endraw %}
</details>

## Testing

The test coverage is quite straightforward and comprehensive.

E.g. [finding the root node](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag_test.go#L23-L36):
```go
func TestAcyclicGraphRoot(t *testing.T) {
	var g AcyclicGraph
	g.Add(1)
	g.Add(2)
	g.Add(3)
	g.Connect(BasicEdge(3, 2))
	g.Connect(BasicEdge(3, 1))

	if root, err := g.Root(); err != nil {
		t.Fatalf("err: %s", err)
	} else if root != 3 {
		t.Fatalf("bad: %#v", root)
	}
}

// ...
func TestAcyclicGraphRoot_cycle(t *testing.T) {
    // ...
}

// ...
func TestAcyclicGraphRoot_multiple(t *testing.T) {
    // ...
}
```

See more tests in [dag_test.go](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/dag_test.go), [graph_test.go](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/graph_test.go), [tarjan_test.go](https://github.com/hashicorp/terraform/blob/72a7c953535b1d7f4dadf39649da3f5563ad5354/internal/dag/tarjan_test.go).

## References

* [GitHub Repo](https://github.com/hashicorp/terraform)
* [Product website](https://www.terraform.io/)
* [Documentation](https://www.terraform.io/docs/)
* [Applying Graph Theory to Infrastructure as Code (Video)](https://www.youtube.com/watch?v=Ce3RNfRbdZ0)

## Copyright notice

Terraform is developed by [HashiCorp](https://www.hashicorp.com/).

Terraform is licensed under the [Mozilla Public License 2.0](https://github.com/hashicorp/terraform/blob/main/LICENSE).
