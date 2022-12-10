+++
title = "Improving Network Efficiency through Sparsification of Network Graphs"
date = 2022-12-09T00:00:00+00:00

[taxonomies]
tags = ["Post"]

[extra]
author = "Abhijeet, Dante, Deepak, Reed, and Zicheng"
+++

# Background

A computer network can be represented as an undirected graph, where switches and end hosts are nodes and connections enabling data transfer between them are edges. An autonomous system (AS) is a group of connected IP networks that are operated by a single entity, which consists of a lot of switches and end hosts. ASes are interconnected to enable communication between networks. As networks grow, there will be more switches and end hosts within autonomous systems with either wire or wireless connection, and more ASes are registered and interconnected with others with physical links.

Redundant links are sometimes introduced into the network to provide more bandwidth or robustness over failure, but this is pretty expensive and requires routers to have more resources to deal with routing information. We would like to see if we can identify important links in the network, and explore the potential of higher efficiency by using fewer redundant links.

![Project Summary](project_summary.png)

# How?

We would like to use graph sparsification methods to remove edges from existing networks, and use various metrics to identify if the sparsified network graph:

- **Edge count**: Direct metric for measuring how aggressively sparsified a graph is.
- **Diameter**: Distance between the furthest pair of nodes, measured by assigning each edge a length of 1.
- **Node and edge connectivity**: The minimum number of nodes or edges whose removal is required to disconnect the graph. Measures redundancy.
- **Connectedness**: Whether each node is reachable from each other node.
- **Longest-shortest max flow**: how much data can be sent through it at any given time.
- **Average max flow**: While Longest-Shortest Max Flow measures worst-case bandwidth, this metric aims to capture average bandwidth.
- **PageRank**: The relative importance of different nodes in a graph by whether they have edges from other important nodes.
- **MST (Minimum spanning tree) runtime**: The time to perform Kruskal's algorithm on the graph.
- **Community structure**: Used Louvain community detection algorithm to identify small, densely connected clusters.

We've experimented with a lot of sparsification methods, both connectivity-preserving and non-connectivity-preserving:

- Spielman Srivastava
- Forest fire
- Local degree
- Local similarity
- Random edge
- Random node edge
- Spanner (preserves connectivity)
- Simmelian overlap

# What about datasets?

We would like to see the how graph sparsification methods work on a real network, so we mainly used two datasets:

- [CAIDA AS relationship graph](https://www.caida.org/catalog/datasets/as-relationships/): This is CAIDA's ongoing project since 1998 that measures the Internet's inter-domain structure.
- [Fat-tree](https://www.cs.cornell.edu/courses/cs5413/2014fa/lectures/08-fattree.pdf): A network topology used in data centers that scales well, with redundancy and high east-west traffic bandwidth. The code for generating K-ary fat-tree is relatively straightforward.
  ```python
  def generate_fattree(k):
      graph = {} # Representing as an adjacency list
      halfK = k // 2
      # Core switches: id's 0 - (k / 2) ^ 2
      for i in range(halfK * halfK):
          graph[i] = []
      # Now add in the nodes for every pod
      nextNode = halfK * halfK
      for pod in range(k):
          # Aggregate switches make up the next k / 2
          # After that, edge switches
          for agg in range(halfK):
              graph[agg + nextNode] = []
              # Add the links from core switches
              for core in range(halfK):
                  graph[agg * halfK + core].append(agg + nextNode)
              # Add the links to edge switches
              for edge in range(halfK):
                  graph[agg + nextNode].append(edge + nextNode + halfK)
          nextNode = nextNode + halfK * 2
      return graph
  ```

Stanford's [AS-733](https://snap.stanford.edu/data/as-733.html) graph was also used during the testing phase.

## … And Bandwidth?

Bandwidth for fat-tree links should be pretty easy to estimate, but this is not the case for links between ASes. We initially plans to utilize [PeeringDB](https://www.peeringdb.com/)'s degree and traffic level to estimate the bandwidth (with some simple machine learning technique), but it turns out that the result is pretty inaccurate:

- The type of business (Video Streaming / CDN) would affect the characteristic of traffic even if the nodes have the same degree;
- The dataset has only categorial data making methods like linear regression ineffective;
- The data from PeeringDB is not accurate/up-to-date and a lot of entities chose to not disclose their level of traffic, and abandoned AS also got mixed in.

![AS Degree vs Traffic Level](as_degree_vs_traffic_level.png)

We end up using degree directly to estimate the traffic of links. Nodes with higher degree indicate that they are likely higher-tier provider ASes, while low-degree nodes are likely low-tier ASes with low traffic level. The rule is described as follows: for AS **A** with degree **a** and AS **B** with degree **b**, the edge between **A** and **B** has `a < b ? a : b` bandwidth. However, this method also has some limitation: some Cloud / VPS provider may provide free peering with customers buying VPS service from them, and degree of this provider AS might be high while customers doesn't really generate a lot of traffic. This would make provider's AS has high degree, which may result in in accurate bandwidth estimation.

Python snippet for loading a graph and assign capacities:

```python
def load_net(file, assign_capacities=False):
    graph = None
    with open(file, 'r') as f:
        graph = nx.Graph()
        for line in f:
            if line.startswith('#'):
                continue

            if "|" in line:
                line = line.split("|")
            else:
                line = line.split()

            if line[0] != line[1]:
                graph.add_edge(int(line[0]), int(line[1]))

    if assign_capacities:
        node_deg = graph.degree()
        node_deg_dict = {}
        for node, deg in node_deg:
            node_deg_dict[node] = deg
        for u, v in graph.edges():
            u_deg = node_deg_dict[u]
            v_deg = node_deg_dict[v]
            graph.edges[u, v]['capacity'] = v_deg if u_deg > v_deg else u_deg

    return graph
```

# Experimental Pipeline

Graphs are read in to the NetworkX library and analyzed in a Jupyter Notebook. 

```python
graphs_init = load_graphs(files, test_path, True)
```

Metrics functions allow us to measure properties of the resulting graphs. For example, this function calculates the longest shortest path endpoints and length (diameter).

```python
def long_shrt_path(g):
    shortest_path_iter = nx.all_pairs_dijkstra_path_length(g)
    max_dist = -1
    s_t_pair = (None, None)
    for s_tdict_pair in shortest_path_iter:
        for t, t_dist in s_tdict_pair[1].items():
            if t_dist > max_dist:
                max_dist = t_dist
                s_t_pair = (s_tdict_pair[0], t)
    return s_t_pair, max_dist
```

We evaluate these metrics on the initial graphs in this first notebook.

```python
metrics_init = calc_metrics(graphs_init)
```

Then, each sparsification method is run in its own notebook to generate sparsified graphs at a variety of parameter settings and compare the results to the initial graph. 

```python
graphs_spanner.loc[0, files] = graphs_init
for f in files:
    for i, stretch in enumerate(graphs_spanner[parameter_name][1:]):
        graphs_spanner.at[i+1, f] = sparsifier_g1(graphs_init[f], sparse_method_name, {parameter_name: stretch})
```

The metrics are recomputed on these sparsified graphs.

```python
for i in graphs_sparse.index[1:]:
    metrics_sparse.loc[i, files] = calc_metrics(graphs_sparse.loc[i, files])
return metrics_sparse
```

Finally, the results of the series of sparsification parameters for a given method are plotted.

```python
def lineplot(data, metricName, paramName, infile, outfile, title, ylabel, sparseMethodName):
    plt.figure()
    pts = [((1 - md["e_ct"]/data.at[0, infile]["e_ct"])*100, md[metricName]) for md in data[infile]]
    pts.sort(key = lambda x:x[0])
    plt.plot([p[0] for p in pts], [p[1] for p in pts], 'o--')
    plt.title(f"{title} vs Sparsity on {f} {sparseMethodName}")
    plt.xlabel("edges removed (%)")
    plt.ylabel(ylabel)
    plt.savefig(os.path.join(figures_path, f"{outfile}_line_{f}_{sparseMethodName}.png"))
```

# Some Interesting Findings

- Longest-shortest max flow could be impacted severely or not at all because it is affected by only a single path. Methods such as local degree and local similarity reduced this flow roughly proportional to the edges removed. This relationship was surprising because the majority of edges in the graph do not directly influence this flow. Other methods such as random edge and random node edge caused a greater impact on this metric than on edge count. Spanner, which preserves connectedness, was especially detrimental, providing an example of a disadvantage for this otherwise promising method. Interestingly, no method reduced this metric to a lesser extent than edge count. This is expected to be possible if a method were to identify this worst-case bandwidth path and avoid removing its edges.
  ![Project Summary](long_shrt_max_flow_line_ft48.txt_local_degree.png)
  ![Project Summary](long_shrt_max_flow_line_ft48.txt_spanner.png)

- Community structure was not preserved in general. The best results on any sparsified graphs were for random edge and scan on the AS datasets, but even then it was only for the gentlest settings that only removed 5 of the edges.
  ![Community Structure](cluster_1.png)

# Conclusions

In this study, we found that sparsification methods were able to significantly reduce the number of edges in computer networks. However, this reduction came at the expense of performance-critical metrics. Therefore, the development of sparsification methods that prioritize network performance could be beneficial for future research in this area. Of the methods tested, only spanner preserved connectedness. Methods that specifically avoid removing edges that represent essential connections or flow bottlenecks could improve upon the results of this work.

In future work, we plan to model the computer networks as directed graphs and conduct similar studies. This would be an interesting problem from a graph theory perspective, as the literature on sparsification algorithms for directed graphs is currently limited. Additionally, we plan to virtualize the networks studied here and simulate network traffic to gain a better understanding of the impact of sparsification in real-world scenarios.
