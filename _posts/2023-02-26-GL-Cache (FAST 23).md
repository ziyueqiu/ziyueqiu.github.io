---
layout:     post
title:      GL-Cache (FAST 23) 论文阅读
date:       2023-02-26
tags:
    - Cache
    - ML
categories: paperreading
comments: true
---

[GL-Cache: Group-level learning for efficient and high-performance caching](https://www.usenix.org/system/files/fast23-yang.pdf)

## Introduction

"learned caches": employed machine learning to improve cache evictions
- object-level learning (e.g. LRB): learns the next access time for each object using dozens of object features and evicts the object with the furthest predicted request time
  - significant computation and storage overheads
  - Zipf distributions "most objects only get a limited number of requests. This leads to limited object-level information for learning"
- learning-from-distribution (e.g. LHD): models request probability distributions to inform eviction decisions
  - 必须 random sample，利用的信息有限，绝大部分 object 在 distribution 的尾部、信息很少
- learning-from-simple-experts (e.g. LeCaR and Cacheus): performs evictions by choosing eviction candidates recommended by experts (e.g., LRU and LFU), and updates experts’ weights based on their past performance on the workload
  - A delay exists between a bad eviction and an update on the expert’s weight (“delayed rewards” in reinforcement learning).

Group-level Learned Cache:
- cluster similar objects into groups using write time
- evict the least useful groups using a merge-based eviction
- introduce a group utility function to rank groups

## Background and motivation

Questions:

- "Object-level learning needs to sample objects and perform inference at each write (eviction). For example, LRB samples 32 objects and copies their features to a matrix for inference for each eviction. In our measurement, each evic- tion (including feature copy, inference, and ranking) takes **200 μs** on one CPU core, indicating that the cache can evict at most 5,000 objects on a single core per second." 200 us 这实验是怎么做的，这数字怎么这么大

## GL-Cache: Group-level learned cache

- Grouping by write time
  - PS: allow an efficient implementation using a log-structured cache

{% include figure.html path="assets/img/fig/GLCache-fig1.png" class="img-fluid rounded z-depth-1" zoomable=false %}

