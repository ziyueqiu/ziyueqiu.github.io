---
layout:     post
title:      Dynacache (HotCloud 15) 论文阅读
date:       2023-01-10
tags:
    - Cache
categories: paperreading
comments: true
---

[Dynacache: Dynamic Cloud Caching](https://www.usenix.org/conference/hotcloud15/workshop-program/presentation/cidon)

## Abstract

- Dynacache, a cache controller, significantly improves the hit rate of web applications, by profiling applications and dynamically tailoring memory resources and eviction policies.
- Dynacache allows Memcached operators to better plan their resource allocation and manage server costs, by estimating the cost of cache hits as a function of memory.

总的来说就是 profiling+tuning 很有提升空间、是值得的，Dynacache 通过更好地平衡 slab 之间的分配来优化性能。

## Introduction

很多人都不觉得 simplicity 和 performance 也是个重要的 trade-off，这里提到了我还挺惊喜的："While a dynamic application-aware cache is conceptually appealing, is it worth the additional complexity in practice?"

{% include figure.html path="assets/img/fig/Dynacache-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

## MEMCACHIER TRACE ANALYSIS

## Design

### Practical Profiling

没看懂：

"Dynacache uses a bucketing scheme similar to Mimir. In this bucketing scheme, instead of keeping track of a shadow eviction queue, there is a linked list of buckets, each containing a fixed number of items. Each incoming request enters the top bucket, and when the top bucket is filled, we remove the bucket at the end of the queue. We maintain a hash function that maps each item to the bucket in which it is stored. We can estimate the stack distance of an in- coming request, by summing the size of all buckets that appear in the queue before it, and adding it to the size of its own bucket divided by 2. This stack distance computation algorithm is much faster than the naïve method, since its complexity is O(B), where B is the number of buckets. For Memcachier, we utilized 100 buckets with 100 keys each."

## 评论

- 感觉跟 LAMA 撞了，而且总体上被 LAMA 高级替代了