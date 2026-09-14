---
layout: post
comments: false
title: "Strongly Connected Components"
excerpt: "Finding the mutually reachable regions hidden inside a directed graph"
date: 2026-08-20 00:00:00
mathjax: true
---

A directed graph can look complicated because every edge has a direction. One useful way to make it simpler is to group vertices that can all reach one another. These groups are called **strongly connected components**, or SCCs.

SCCs appear in dependency analysis, build systems, control-flow graphs, web crawls, social networks, and state machines. The key idea is simple, but it turns a graph with many cycles into a collection of well-defined regions.

## Mutual reachability

Let \\(G = (V, E)\\) be a directed graph. Two vertices \\(u\\) and \\(v\\) are strongly connected when:

1. there is a directed path from \\(u\\) to \\(v\\), and
2. there is a directed path from \\(v\\) to \\(u\\).

This is stronger than merely being in the same connected part of the graph. In an undirected graph, an edge can be traversed in either direction. In a directed graph, we must respect every arrow.

For example, consider the edges:

```text
0 → 1, 1 → 2, 2 → 0
2 → 3, 3 → 4, 4 → 3
```

Vertices `{0, 1, 2}` form one SCC: every vertex can reach the other two. Vertices `{3, 4}` form another. There is a path from the first component to the second, but no path back, so all five vertices do not belong to one component. A single vertex is also an SCC. It does not need a self-loop: a vertex can reach itself using a path of length zero. Strong connectivity is an equivalence relation. Every vertex belongs to exactly one SCC, and two different SCCs never overlap. This gives us a partition of the graph. We can then compress each SCC into one “super-vertex.” If an original edge goes from a vertex in component \\(A\\) to a vertex in component \\(B\\), we add an edge \\(A \\rightarrow B\\) to the compressed graph. The resulting graph is called the **condensation graph**. It is always a directed acyclic graph (DAG). If the condensation graph had a cycle, all components on that cycle could reach one another, so they should have been one larger SCC.

This is often the most important consequence of SCC decomposition:

```text
arbitrary directed graph
        ↓ find SCCs
condensation DAG
        ↓
topological ordering, dynamic programming, dependency analysis
```

## Kosaraju’s algorithm

One approachable linear-time algorithm is **Kosaraju’s algorithm**. It performs two depth-first searches:

1. Run DFS on the original graph and record vertices when their search finishes.
2. Reverse every edge.
3. Process vertices in decreasing finishing-time order, running DFS on the reversed graph. Each DFS traversal discovers exactly one SCC.

The edge reversal is important. A path \\(u \\leadsto v\\) in the original graph becomes a path \\(v \\leadsto u\\) in the reversed graph. The finishing order from the first traversal tells us which component should be explored first during the second traversal.

### A small example

Suppose the original graph has two SCCs, \\(A\\) and \\(B\\), with an edge \\(A \\rightarrow B\\). The condensation graph is:

```text
A → B
```

After reversing edges, it becomes:

```text
A ← B
```

The first DFS gives the component that can lead outward the largest finishing time. When the second DFS starts from that vertex in the reversed graph, it cannot leak from one SCC into another in the wrong direction. It collects exactly one component before the next starting vertex is selected.

## Python implementation

The implementation below uses adjacency lists and returns a list of components. The component IDs are arbitrary; what matters is which vertices occur together.

```python
def strongly_connected_components(graph):
    """Return the SCCs of a directed graph as lists of vertices.

    graph[u] contains every vertex v for which u -> v exists.
    Vertices are numbered from 0 through len(graph) - 1.
    """
    n = len(graph)
    reversed_graph = [[] for _ in range(n)]

    for u in range(n):
        for v in graph[u]:
            reversed_graph[v].append(u)

    visited = [False] * n
    finish_order = []

    def first_dfs(u):
        visited[u] = True
        for v in graph[u]:
            if not visited[v]:
                first_dfs(v)
        finish_order.append(u)

    for u in range(n):
        if not visited[u]:
            first_dfs(u)

    visited = [False] * n
    components = []

    def second_dfs(u, component):
        visited[u] = True
        component.append(u)
        for v in reversed_graph[u]:
            if not visited[v]:
                second_dfs(v, component)

    for u in reversed(finish_order):
        if not visited[u]:
            component = []
            second_dfs(u, component)
            components.append(component)

    return components
```

For the earlier example:

```python
graph = [
    [1],       # 0 -> 1
    [2],       # 1 -> 2
    [0, 3],    # 2 -> 0, 3
    [4],       # 3 -> 4
    [3],       # 4 -> 3
]

print(strongly_connected_components(graph))
# [[0, 2, 1], [3, 4]]  (ordering may vary)
```

The algorithm visits every vertex and every edge a constant number of times. Its time complexity is therefore

$$
O(|V| + |E|),
$$

and its space complexity is also \\(O(|V| + |E|)\\), including the reversed graph.

## C++ implementation

Here is the same algorithm in C++. The first DFS records finishing order, and the
second DFS explores the reversed graph in the opposite order.

```cpp
#include <algorithm>
#include <iostream>
#include <utility>
#include <vector>

using Graph = std::vector<std::vector<int>>;

void dfs_order(int u, const Graph& graph, std::vector<bool>& visited,
               std::vector<int>& order) {
    visited[u] = true;

    for (int v : graph[u]) {
        if (!visited[v]) {
            dfs_order(v, graph, visited, order);
        }
    }

    order.push_back(u);
}

void dfs_component(int u, const Graph& reversed_graph,
                   std::vector<bool>& visited,
                   std::vector<int>& component) {
    visited[u] = true;
    component.push_back(u);

    for (int v : reversed_graph[u]) {
        if (!visited[v]) {
            dfs_component(v, reversed_graph, visited, component);
        }
    }
}

std::vector<std::vector<int>> strongly_connected_components(
    const Graph& graph) {
    const int n = static_cast<int>(graph.size());
    Graph reversed_graph(n);

    for (int u = 0; u < n; ++u) {
        for (int v : graph[u]) {
            reversed_graph[v].push_back(u);
        }
    }

    std::vector<bool> visited(n, false);
    std::vector<int> finish_order;
    finish_order.reserve(n);

    for (int u = 0; u < n; ++u) {
        if (!visited[u]) {
            dfs_order(u, graph, visited, finish_order);
        }
    }

    std::fill(visited.begin(), visited.end(), false);
    std::vector<std::vector<int>> components;

    for (auto it = finish_order.rbegin(); it != finish_order.rend(); ++it) {
        if (!visited[*it]) {
            std::vector<int> component;
            dfs_component(*it, reversed_graph, visited, component);
            components.push_back(std::move(component));
        }
    }

    return components;
}

int main() {
    Graph graph = {
        {1},       // 0 -> 1
        {2},       // 1 -> 2
        {0, 3},    // 2 -> 0, 3
        {4},       // 3 -> 4
        {3},       // 4 -> 3
    };

    const auto components = strongly_connected_components(graph);

    for (const auto& component : components) {
        for (int vertex : component) {
            std::cout << vertex << ' ';
        }
        std::cout << '\n';
    }
}
```

One possible output is:

```text
0 2 1
3 4
```

The order of vertices inside a component, and sometimes the order of the
components themselves, can vary without changing the result.

## Why the second DFS finds one component

The correctness comes from the relationship between finishing times and the condensation DAG.

Consider a component (C). During the first DFS, either the search finishes all reachable components after visiting (C), or it enters (C) from somewhere else. In both cases, the component that is “later” in the direction of the condensation DAG receives the larger finishing time.

Therefore, when we process vertices in decreasing finishing order, we start with a component that has no outgoing edge to an unprocessed component in the reversed graph. The second DFS can move freely inside the chosen SCC, but it cannot cross into another unprocessed SCC. It consequently returns exactly one component.

Repeating this argument removes one SCC at a time until every vertex has been assigned.

## Tarjan’s alternative

Kosaraju is easy to explain, but it needs two graph representations and two DFS passes. **Tarjan’s algorithm** finds all SCCs in one DFS of the original graph.

Tarjan assigns each vertex:

- an `index`, representing when it was discovered;
- a `low_link` value, representing the smallest discovery index reachable from it while staying connected to the current DFS structure.

Vertices are kept on a stack until their component is complete. When a vertex has `low_link[u] == index[u]`, it is the root of an SCC, and vertices are popped from the stack until that root is reached.

Tarjan also runs in \\(O(|V| + |E|)\\) time and space. In practice, either algorithm is a good choice. Kosaraju can be more approachable when building a first implementation; Tarjan is convenient when an extra reversed graph or an additional traversal is undesirable.

## Common mistakes

There are a few details that frequently cause bugs:

- **Using an undirected traversal.** SCCs require reachability in both directed directions, not just membership in one undirected region.
- **Forgetting disconnected vertices.** The outer DFS loop must start a traversal from every unvisited vertex.
- **Using the wrong finishing order.** Kosaraju’s second pass processes vertices in decreasing finishing time.
- **Ignoring recursion depth.** A graph that is essentially one long chain can exceed Python’s recursion limit. An iterative DFS or an increased recursion limit may be needed for large inputs.
- **Confusing SCCs with cycles.** A component can contain one vertex without a self-loop, and a component may contain many overlapping cycles.

## The mental model

An SCC is a region of a directed graph where movement is reversible at the level of reachability: from anywhere inside the region, we can eventually get to anywhere else inside it.

Once those regions are compressed, every remaining edge points between components and the cycles disappear. That transformation—from a cyclic directed graph to a DAG of mutually reachable regions—is why SCCs are such a useful building block for graph algorithms.
