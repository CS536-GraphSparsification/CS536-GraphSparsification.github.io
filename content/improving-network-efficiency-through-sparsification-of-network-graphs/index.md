+++
title = "Improving Network Efficiency through Sparsification of Network Graphs"
date = 2022-12-09T00:00:00+00:00

[taxonomies]
tags = ["Post"]

[extra]
author = "Abhijeet, Dante, Deepak, Reed, and Zicheng"
+++

# Background

A computer network can be represented as an undirected graph, where switches and end hosts are nodes and connections enabling data transfer between them are edges. An autonomous system (AS) is a group of connected IP networks that are operated by a single entity, which consists of a lot of switches and end hosts. ASes are interconnected to enable communication between networks. As networks grow, there will be more switches and end hosts within autonomous systems with either wire or wireless connection, and more ASes are registered and interconnected with others with physical links. Redundant links are sometimes introduced into the network to provide more bandwidth or robustness over failure, but this is sometimes pretty expensive and requires routers to have more resources to deal with routing information. We would like to see if we can identify important links in the network, and explore the potential of higher efficiency by using fewer redundant links.

# How?

We need some network graphs.
