---
layout:     post
title:      CloudCache (FAST 16) 论文阅读 - Reuse Working Set (RSS) admission
date:       2022-10-29
tags:
    - Cloud
    - Elasiticity
    - Cache
categories: paperreading
comments: true
---

[CloudCache: CloudCache: On-demand Flash Cache Management for Cloud Computing](https://www.usenix.org/conference/fast16/technical-sessions/presentation/arteaga)

我关心 caching for cloud computing 有什么需要在意的，但是这篇主要是关注 Flash cache，而且还是 private cloud。

## Abstract

目的是：

- meet VM cache demands
- minimize cache wear-out

方法是：

- propose a new cache demand model "Reuse Working Set (RSS)" **to capture only the data with good temporal locality**
- and use RSS size (RSSS) to model a workload's *cache demand*
- **predict RWSS online** and admit only RWS into the cache
- propose a dynamic cache migration approach to balance cache load across hosts by live migrating cached data along with the VMs (both on-demand migration of dirty data and background migration of RWS)
- Support rate limiting on transfer to limit impact to **co-hosted VMs**
- Evaluate on **real-world traces**

看到这里有的 Questions:

- ~~如何识别 "data with good temporal locality"~~ 这里的意思就是 RWS
- ~~什么是 "cache demand"~~ WS/RWS
- "online prediction" 开销如何
- ~~migration 具体怎么做~~
- ~~什么是 "impact to co-hosted VMs"~~ 因为要向外 migrate 数据到 co-hosted (Flash write)，所以有可能对 co-hosted 的性能有负面影响
- ~~什么样的 "real-world traces"~~ several-month traces from their prior work

## Introduction

新的 takeaway 是：

- the amount of flash cache that can be employed on a host is limited -> 所以准入策略是重要的
- 但如果数据就是都太重要了呢？migration！

## Motivation

Questions:

- ~~what does "consolidated systems" mean?~~ 可能是会有多个 VM 在一个 host

## Architecture

{% include figure.html path="assets/img/fig/CloudCache-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Every host employs a flash cache, shared by the local VMs, and every VM’s access to its remote disk goes through this cache. A VM disk is rarely write-shared by multiple hosts (beyond this paper).

CloudCache supports different write caching policies: (1) Write-invalidate; (2) Write-through; (3) Write-back

{% include figure.html path="assets/img/fig/CloudCache-fig2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Questions:

- SAN/IP-SAN 什么的完全不懂

## On-demand Cache Allocation

### RWS-based Cache Demand Model

Working Set, WS(t, T) at time t is defined as the set of distinct (address-wise) data blocks referenced by the workload during a time interval [t-T, t].

->

Reuse Working Set, RWS_N (t, T), which is defined as the set of distinct (address-wise) data blocks that a workload has reused at least N times during a time interval [t−T,t].

对于本文的 workload in Table 1 以及 MSR Cambridge traces，N = 1 或者 2 才是最好的。（传统 cache algorithm N = 0）

如果要使用 RWS 模型，需要解决两个新问题：

- How to track window?
  - real-time based window (x number of accesses made by the process，因为可能 VM 比较空闲的时候，非常长时间用不到 cache，这个 window 一直不切换，高估了 cache demand 也没有及时调整)
- How to decide the size of the time window?
  - 太大可能放了太多过去的情况进入考虑，太小可能低估了 cache demand
  - profile 一段时间，根据绘制出来的图，选择 "knee point" size，如果没有 "knee point"，就选相对小的，下图例子应该选一个 24-48h 之间的数字

{% include figure.html path="assets/img/fig/CloudCache-fig3.png" class="img-fluid rounded z-depth-1" zoomable=true %}

### Online Cache Demand Prediction

要点如下：

- The success of RWSS-based cache allocation also depends on whether we can accurately **predict** the cache demand of the next time window **based on the RWSS values observed from the previous windows**.
- 具体方法：To address this problem, we consider the *classic exponential smoothing and double exponential smoothing methods*. The former requires a smoothing parameter α, and the latter requires an additional trending parameter β . The values of these parameters can have a significant impact on the predic- tion accuracy. We address this issue by using the self tuning versions of these prediction models, which estimate these parameters based on the error between the predicted and observed RWSS values.
- 下图是一个例子，值得提醒的是这个图我看了很久才完全理解对：首先 (a) 图的 WSS 曲线的几个 peak 都来自大型 scanning；其次 (b) 图为什么即使是 WSS 都数值不一样了呢？因为整个 (b) 图都是用了 smoothing method 的；(b) 图还想要说明的是 RWSS 其实有一些 "occasional bursts of IOs that do not reflect the general trend of the workload"，用 "RWSS+Filter" 效果更好，也确实可以看到一些小的凸起比如 day5 就没了。

{% include figure.html path="assets/img/fig/CloudCache-fig4.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Questions:

- "exponential smoothing methods" 引入得好突然，不是很懂选它的前因后果
- 虽然 RWSS+Filter 确实把突变点抹平了，但我不理解的是，为什么 occasional bursts of IOs 不值得这一天给他多分一点，确实这一天它有更多的需求啊，甚至不是那种来自 scanning 的需求（RWSS 已经过滤掉了只 hit 一次的情况）
- 不懂这个 window 实际是怎么进行的：是缓慢滑行（pop & push 那种感觉），还是单独 window 考虑（有提到用了 circular buffer）；这部分会不会影响性能？

### Cache Allocation and Admission

- The allocation of cache capacity should not incur costly data copying or flushing.
  - Hence, we consider **replacement-time enforcement** of cache allocation with logical partitioning at replacement time: a VM that has not used up its allocated share takes its space back by replacing a block from VMs that have exceeded their shares.
  - If the cache is not full, the spare capacity can be allocated to the VMs proportionally to their predicted RWSSes or left idle to reduce wear-out.
- Hybrid stage policy (address staging + data staging) **in main memory**: 需要考虑 RWS 或者 WS 都需要存历史信息；只存 address 的好处是不占空间 (8B address per 4KB data)，如果存 data 虽然挤占了存 address 的空间，但可以补偿一些 second miss —— 这里有个 trade-off

Questions:

- 不知道这里的 LRU 是怎么维护的，我好像 whole picture 不清晰
- in-memory 会不会价值更高？

### Evaluation

- CloudCache is created upon block-level virtualization by providing virtual block devices to VMs and transparently caching their data accesses to remote block devices accessed across the network (Figure 1).
  - It includes **a kernel module** that implements the virtual block devices, monitors VM IOs, and enforces cache allocation and admission, and a user-space component that measures and predicts RWSS and determines the cache shares for the VMs.
  - The kernel module **stores the recently observed IOs in a small circular buffer** for the user-space component to use, while the latter informs the former about the cache allocation decisions. 

#### Prediction Accuracy

#### Staging Strategies

#### WSS vs. RWSS-based Cache Allocation

{% include figure.html path="assets/img/fig/CloudCache-fig5.png" class="img-fluid rounded z-depth-1" zoomable=true %}

No allocation = workload can use up the entire cache where the cache is large enough to hold the entire working set and does not need any replacement

WSS 看上去也准入了 first miss，但比 No-Alloc 小很多是因为多了时间窗（比如 1 天的约束）。

Questions:

- **怎么做到 No alloc 和 RWSS 明明 hit ratio 有一定差距，latency 却差不多的？**

## Dynamic Cache Migration

### Live Cache Migration

= move some VMs to other hosts

结合以下两种方法：

- On-Demand Migration
- Background Migration

Questions:

- 怎么把原 workload 分到实验配置的两个 host 之上的？