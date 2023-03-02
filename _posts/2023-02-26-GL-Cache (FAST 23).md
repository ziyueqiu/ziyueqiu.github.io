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

- Learning object-group utility
  - gradient boosting machines (GBM) because tree models do not require feature normalization
  - formulate the learning task as a regression problem that minimizes the mean square loss (L2) of object-group utilities
  - A sampled group may be evicted before being used for training. Such evictions halt the tracking of group utility. GL-Cache keeps ghost entries for objects which have not been factored into group utility.
  - 一次扔掉最差的 group 附近的 $$N_{group}$$ 个 group，这个参数可能设置成整体的 1% 之类的

Questions:

- 如果有好几个 application 同时写入，凭什么说有相似的 write-time 的 object 相似呢？
- "GL-Cache generates new training data by sampling cached object groups, and it copies the features of the sampled groups into a pre-allocated memory region" 这不是也有 random sampling (v.s. learning-from-distribution)?

## Evaluation

放两个有意思的图：

{% include figure.html path="assets/img/fig/GLCache-fig2.png" class="img-fluid rounded z-depth-1" zoomable=false %}

{% include figure.html path="assets/img/fig/GLCache-fig3.png" class="img-fluid rounded z-depth-1" zoomable=false %}