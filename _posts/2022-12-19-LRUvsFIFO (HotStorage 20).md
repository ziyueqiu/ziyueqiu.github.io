---
layout:     post
title:      LRUvsFIFO (HotStorage 20) 论文阅读
date:       2022-12-19
tags:
    - Cloud
    - Cache
categories: paperreading
comments: true
---

[It's Time to Revisit LRU vs. FIFO](https://www.usenix.org/conference/hotstorage20/presentation/eytan)

## Abstract

Two new trends:

- new caches such as front-ends to cloud storage have very large scales and this makes managing cache metadata in RAM no longer feasible
- new workloads have emerged that possess different characteristics.

## Introduction

LRU vs. FIFO

- "Our study looks at such a setting for a cloud object storage service. An object storage service stores **large, immutable objects in the cloud using a RESTful API**. Since it services customers over the web, it typically suffers from **poor latency** and as such, **holding a front-end edge cache** can be very beneficial."
- FIFO’s main merit is its management simplicity: hold all the items in the cache in a queue and evict the next in line, with no need to update information about items already in the queue.

Revisit LRU vs. FIFO

- A New Scale to Caches: 以往的情况是 cache metadata 相比 cache size 只是一小部分，但是目前的视角转换为云上的 cache size 可以非常大，这样 cache metadata 的绝对大小也会非常大（尤其是 LRU 这种实现），no longer fit in memory
and need to spill over to the persistent media
- New Workloads: Caching evaluations carried out in the past have centered around workloads for **memory, files and block storage**. New workloads, such as **big data analytics and machine learning**, have different characteristics that may significantly skew old results.

Contribution（部分）

- IBM Cloud Object Storage service (COS): weekly traces of 99 tenants, accounting for over 850 Million I/O requests and amounting to 158 TBs of data being accessed

## Large Caches and Cost Model

### The Effect of Large Scale Cache Deployment on Cache Management

- Once metadata is not held entirely in RAM, it is very likely that metadata updates would entail relatively expensive updates to persistent disk. FIFO is a simple heuristic that attempts to approximate LRU to the best of its ability. It is unique in that it is hardly affected when the cache metadata does not fit in memory.
- We note that we have experimented with several different caching algorithms and found that on the diverse workloads that we run, **none of the algorithms consistently outperform LRU in terms of hit rate**(其实这并不能说明什么，也许反过来也成立). Thus LRU serves as a good cache eviction policy as any.

{% include figure.html path="assets/img/fig/LRUvsFIFO-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

### A Cache Cost Model

{% include figure.html path="assets/img/fig/LRUvsFIFO-fig2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

## Traces

The IBM COS trace

- IBM Cloud Object Store is a public cloud based object storage service for storing immutable objects, primarily via a RESTful API.
- We collected 99 traces from this service, each comprised of all the data access requests issued over **a single week** by a **single tenant** of the service. The traces include **PUT, GET, and DELETE** objects requests and include object sizes along with obfuscated object names.
- We were able to identify some of the workloads as SQL queries, Deep Learning workloads, Natural Language Processing (NLP), Apache Spark data analytic, and document and media servers. But many of the workloads’ types remain unknown.
- The access patterns are very diverse and the object sizes also show great variance.
- In our simulation we break large objects into fixed size 4MB blocks and treat each one separately at the cache layer.

{% include figure.html path="assets/img/fig/LRUvsFIFO-fig3.png" class="img-fluid rounded z-depth-1" zoomable=true %}

## Evaluation Results

{% include figure.html path="assets/img/fig/LRUvsFIFO-fig4.png" class="img-fluid rounded z-depth-1" zoomable=true %}

{% include figure.html path="assets/img/fig/LRUvsFIFO-fig5.png" class="img-fluid rounded z-depth-1" zoomable=true %}

## Discussion

略

## 评论

虽然是一篇很有趣的工作，尝试做了一个“新鲜“的 revisit，但是我仍有几个疑虑：

- 只比较 LRU 和 FIFO 这两个同一类的算法是不是过于简单了呢？诚然 FIFO 和 LRU 在命中率非常接近的时候，FIFO 可能由于 simplicity 取胜；但事情可能大不一样，当我们引入了其他类型的算法进行对比，比如 2Q，TinyLFU，我很好奇这种所谓的“New Workloads”具体有什么算法倾向性/有没有可能 FIFO 和 LRU 就是菜鸡互啄呢？（后来发现 sec5 discussion 探讨了一点点）
- 关于“A New Scale to Caches”，我不理解为什么 metadata size 太大会放不下，不能 consistent hashing + multiple DRAM servers 解决吗？也许有更公平的对比的 metrics？
- 看起来本文只考虑了在 persistent storage 作 cache 的情况下，性能公式需要考虑 latency of metadata，但其实 metadata space overhead 也不同，对应的 budget 也会不同：allocation monetary cost 更少，这也是 simplicity 可能带来的好处。

另外，关于 cache metadata overhead 的担忧其实并不是新鲜事，比如 [SegCache](https://www.usenix.org/conference/nsdi21/presentation/yang-juncheng) 也涉及了这方面的讨论和优化。