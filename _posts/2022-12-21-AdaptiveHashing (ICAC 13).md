---
layout:     post
title:      AdaptiveHashing (ICAC 13) 论文阅读
date:       2022-12-21
tags:
    - Cache
categories: paperreading
comments: true
---

[Adaptive Performance-Aware Distributed Memory Caching](https://www.usenix.org/conference/icac13/technical-sessions/presentation/hwang)

这篇非常有意思啊，我之前对 consistent hashing 算是非常一知半解。借由本文学习如何漂亮地增删 node，以及利用 consistent hashing 作更定制化的 adaptive hashing :) 而且本文给了一些经验性的结论 (比如 fix beta = 0.01 就够了）可以直接 follow。

## Abstract

- use a new, adaptive partitioning algorithm that ensures that load is evenly distributed despite variations in object size and popularity（所谓的 despite variation in object size and popularity 是只观测 usage 和 miss rate，对两者加权）
- implement an adaptive hashing system as a proxy and node control framework for memcached ([open-source repo](https://github.com/jinho10/dht-sched))
- evaluate it on EC2（考虑了 one-hour increment charge）

## Introduction

- over-provisoining the caching tier to ensure there is capacity for peak workloads is a poor solution since cache nodes are often expensive, high memory servers.

Contributions:

- A hash space allocation scheme that allows for targeted load shifting between unbalanced servers.
- Adaptive partitioning of the cache’s hash space to automatically meet **hit rate** and **server utilization** goals.
- An automated replica management system that adds or removes cache replicas based on overall cache performance.（关于这个 replica，本文几乎没细说，感觉是 orthogonal 的，就理解为 cache node 吧）

实现：moxi + memcached

## Background and Motivation

Figure 1 illustrates basic operations of a consistent hashing scheme: node allocation, virtual nodes, and replication.

{% include figure.html path="assets/img/fig/AdaptiveHashing-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

下图展示了一些 inbalance 的例子，但是我不理解 (d) 图，如果是 cache size / # of object，为啥 server 0 的柱子这么矮，不应该是 server 8 最小吗？

{% include figure.html path="assets/img/fig/AdaptiveHashing-fig2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

## System Design

Our system has three important phases: initial hash space assignment using virtual nodes, space partitioning, and memory cache server addition/removal.

### System Operation and Assumptions

- The centralized architecture handles all the requests from applications so that it can control hash space in one place which means object distribution can be controlled easily

### Initial Assignment

需要一个初始分配，满足对 n0 个机器，每个机器都可以做到 hash space 和任何其他机器都相邻，也就是一个 n0 * (n0-1) 个不重复的 pair。本文提出了一个没有错误的 O(n3) 算法，但实际上 O(n2) 完全可以做到。

{% include figure.html path="assets/img/fig/AdaptiveHashing-fig3.png" class="img-fluid rounded z-depth-1" zoomable=true %}

### Hash Space Scheduling

A simplified weighted moving average (WMA) with the scheduling time t0 is used to estimate the hit rate smoothly over the scheduling times:

- $$h_i = 1− \frac{set(i)}{(get(i)−set(i)}$$ ("Hit rate" is a composite metric to represent both object sizes and key distribution, and this also applies when servers have different cache size)
- $$r_i(t) = r_i(t)/t_0 + ui/max_{a≤j≤n}$$ {uj}
- c = α·h+(1−α)·r, h is miss rate and r is usage ratio
- GoaL: minimize $$\sum_{i=1}^n c_i$$
- Approach: find a maximum cost $$c_i$$ and a minimum cost $$c_j$$ -> change hash space and migrate objects from $$c_i$$ to $$c_j$$, the step is β·(1−$$\frac{c_j}{c_i}$$)×\|$$s_i$$\|. β 在本文固定为 0.01.

### Node Addition/Removal

{% include figure.html path="assets/img/fig/AdaptiveHashing-fig4.png" class="img-fluid rounded z-depth-1" zoomable=true %}

- Addition 的描述特别迷惑：找出 cost 最不好（最大）的 k 个，每个都拿走一半的 hash space 给新机器。这个“一半”是怎么选出的 magic number，以及如何拿走这 k 个机器的部分，同时满足每个机器都和其他有相邻点，我确实懵了。原文："new erver takes over exactly half of the hash space from $$s_i^∗$$, which is \|si\|/2."
- Removal: $$s_i$$ 得到 $$\frac{c_j}{c_i+c_j}$$x\|s_k\|

### Implementation Considerations

- Data migration: The best way is to migrate the affected data behind the scene when the scheduling decision is made. The load-balancer can control the data migration by getting the data from the previous server and setting the data to the new server. (Couchbase [6], an open source project, currently uses a data migration so that it is already publicly available.)
- Scheduling cost estimation: In the scheduling algorithm, the cost function uses the hit rate and the usage ratio because applications or load-balancers do not know any information (memory size, number of CPUs, and so on) about the attached memory cache servers. Estimating the exact performance of each cache server is challenging, especially under the current memory cache system. However, using the hit rate and the usage ratio makes sense because these two factors can represent the current cache server performance. Therefore, we implement the system as practical as possible to be deployed without any modifications to the existing systems.

## Experimental Evaluation

Figure 7 illustrates the initial hash space assignment. (c) 图我不是很理解，原文："Figure 7(c) shows the hash size distribution across 20 servers-our approach has a less variability in the hash space assigned per node."

{% include figure.html path="assets/img/fig/AdaptiveHashing-fig5.png" class="img-fluid rounded z-depth-1" zoomable=true %}

另外的一些 magic number：

- Since β changes the total scheduling time and determines fluctuation of the results, we fix β as a 0.01 (1%) based on our experience running many times.
- Default scheduling frequency is 1 min.

这是一些实验结果：

{% include figure.html path="assets/img/fig/AdaptiveHashing-fig6.png" class="img-fluid rounded z-depth-1" zoomable=true %}

我觉得从实验结果来看慢慢调确实太慢了，比如 (b) 图的 hit rate 变化，或者 (c) 图的 hash space 花了 50 min+ 才调到相对稳定。我考虑的解决办法可能是根据计算结果一步调到位，再微调。当然，有可能调的慢也没关系，因为这里才展示了 5 个小时的数据，完整的 traces 可以到 100 小时+。

[User Performance Improvement] 触发 node addition/removal 的 "policy" 不能说不好，但是大有改进空间：如果某个机器 overload（也许是 SLA 满足不了）一段时间就会分配一个新的机器，如果某个机器的使用什么的低于某个 threshold 足够久，可能会回收。

## 评论

- 感觉与其用 alpha 调 usage 和 miss rate 比例权重，不如考虑 miss traffic，和 usage * miss rate 接近；
- 与前者同理，有了绝对指标，node addition/removal 也会更有章法。原先的 goal 还涉及到 alpha 的调参，过于间接，有点隔靴搔痒了