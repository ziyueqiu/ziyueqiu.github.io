---
layout:     post
title:      D3N (IEEE Big Data 19) 论文阅读 - multi-layer cache size auto-configuration
date:       2022-11-05
tags:
    - Cloud
    - Elasiticity
    - Cache
categories: paperreading
comments: true
---

[D3N: A multi-layer cache for the rest of us](https://www.ccs.neu.edu/home/pjd/papers/ekaynar_bigdata19.pdf)

networking + cache auto-configuration

## Abstract

- big-data jobs assume high bandwidth 大数据场景需要高带宽
- large network imbalances due to over-subscription and incremental networking upgrades 但是实际上整个网络架构并不如理想，*over-subscription*（比如 input 带宽加起来其实比 output 链路的能力要大很多，但是 networking 会假设 input 不容易全用满）以及网络可能不断加入新的输入（也是说 input 会越来越大 “*organic growth*”）
- 给出来了一个 2-layer D3N cache (on Ceph RADOS Gateway)

## Introduction

D3N = Datacenter-Data-Delivery Network

- D3N uses high-speed storage (e.g. NVMe flash or DRAM) to cache datasets <u>on the access side of links</u> in a hierarchical network, dynamically allocating cache space across layers based on observed workload patterns and link speeds, so that cache capacity is preferentially used for traffic crossing the most over-subscribed links.

Questions:

- "D3N has required no changes to the interfaces of any Ceph services, involves no additional meta-data services (e.g, to locate cached blocks), and all policies are implemented based purely on local information." really?

## Motivation

这是架构图：

{% include figure.html path="assets/img/fig/D3N-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

本文前面提到过 D3N cache 的设计是 2-layer 的，对着架构图来说，实际上就是“intra-rack”一层，”inter-rack“一层。

为什么要这样呢？这里解释了 "differing upgrade schedules and split ownership typically **prevent inter-cluster networks from being upgraded at the same time**, resulting in significant **bandwidth mismatches** across compute clusters." 所以除了需要 cache 来吸收请求，还需要 intra 和 inter cache size 的动态调整，来平衡请求（当然也不是万能的）。

事实上例子也印证了直觉 "the multi-level approach offers better performance than either a pure L1 or pure L2 approach for 4 cache servers or more"：

{% include figure.html path="assets/img/fig/D3N-fig2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

## D3N Architecture

- For simplicity and resiliency, as well as for integration in existing storage solutions, all caching and routing decision are based on local information rather than central coordination.

{% include figure.html path="assets/img/fig/D3N-fig3.png" class="img-fluid rounded z-depth-1" zoomable=true %}

- Each chunk (4 MiB) has **a “home location” within the L2 cache**, and L1 misses are forwarded to the chunk home location. Only in the event of a miss at the home location is a request forwarded to the data lake, the results of which are cached at both the home (i.e. L2) and client-serving (L1) locations. The L1 and L2 caches are unified: L1 requests for a chunk received at that chunk’s home location result in a single cached copy of the data.

- Limitations
  - Local cache management: Since caching decisions are performed locally within the cache pools in each layer, D3N can make globally suboptimal caching decisions. For example, a few popular blocks can be replicated across all the L1 caches flooding the capacity and preventing caching of slightly less popular blocks.
  - Lack of fairness: Compute clusters participating in D3N share resources such as rack space, storage, power, and network bandwidth with D3N. Even though D3N tries to provide a common good by eliminating network bottlenecks, the individual benefits each cluster gets from D3N may be different and disproportionate to the resources they provide.

## Implementation within Ceph

Questions:

- Figure6 没看懂

略

## Evaluation

略

## Related Work

略

