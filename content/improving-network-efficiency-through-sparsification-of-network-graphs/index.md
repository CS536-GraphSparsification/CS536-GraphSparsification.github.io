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

# How?

We would like to use graph sparsification methods to remove edges from existing networks, and use various metrics to identify if the sparsified network graph:

- Edge count: Direct metric for measuring how aggressively sparsified a graph is.
Diameter: Distance between the furthest pair of nodes, measured by assigning each edge a length of 1.
- Node and edge connectivity: The minimum number of nodes or edges whose removal is required to disconnect the graph. Measures redundancy.
- Connectedness: Whether each node is reachable from each other node.
- Longest-shortest max flow: how much data can be sent through it at any given time.
- Average max flow: While Longest-Shortest Max Flow measures worst-case bandwidth, this metric aims to capture average bandwidth.
- PageRank: The relative importance of different nodes in a graph by whether they have edges from other important nodes.
- MST (Minimum spanning tree) runtime: The time to perform Kruskal's algorithm on the graph.
- Community structure: Used Louvain community detection algorithm to identify small, densely connected clusters.

We've experimented a lot of sparsification method, both connectivity-preserving and non-connectivity-preserving:

- Spielman Srivastava
- Forest fire
- Local degree
- Local similarity
- Random edge
- Random node edge
- Spanner (preserves connectivity)
- Simmelian overlap

# What about datasets?
