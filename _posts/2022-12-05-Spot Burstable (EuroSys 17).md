---
layout:     post
title:      Spot Burstable (EuroSys17) 论文阅读
date:       2022-12-05
tags:
    - Cloud
categories: paperreading
comments: true
---

[Exploiting Spot and Burstable Instances for Improving the Cost-efficacy of In-Memory Caches on the Public Cloud](https://qianlin404.github.io/assets/pdf/wangEurosys2017spot.pdf)

搭建 cost-efficient caches，利用 Spot Instance 和 Burstable Instance 是很有意思的问题

另外，这个工作居然没有名字 :(

## Abstract

用 spot instance 存 cold content

用 burstable instance 作 cache 的 backup

还需要 spot instance price prediction

## Introduction and Motivation

Contributions:
- Data Analysis, Optimization, and Algorithm Design: We offer novel ways of combining regular on-demand, spot, and burstable instances advised both by the needs/properties of memcached, as well as the relative strengths and weaknesses of these instance types.
- System Design and Implementation: We offer a complete implementation of a dynamic resource allocation mechanism for memcached and a working prototype on EC2. The core components of our system include a global controller that periodically collects/predicts workload properties and conducts online optimization for resource allocation, a load balancer that dispatches requests to the appropriate nodes based on popularity and carries out failure recovery when spot revocation occurs, and a key partitioner that classifies the keys into hot and cold keys for our hot-cold mixing.
- Evaluation: We present an empirical evaluation of our ideas combining real-world and synthetic workloads as well as simulations and live experiments on our EC2 prototype. Our key results and lessons are: (i) our spot modeling outperforms the commonly used baselines by offering fewer spot revocations, (ii) our hot-cold mixing, informed by our modeling of spot prices, helps improve cost savings by 50-80% compared to only using regular instances, and (iii) our burstable-based backup helps reduce performance degradation during spot failures, e.g., the 95% latency during failure recovery improves by 25% compared
to a backup based on regular instances.

## Background

price model:

{% include figure.html path="assets/img/fig/ExploitSpot-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

略

## 评论

对不起太复杂了，没坚持下来，之后再读细节吧。核心是两个改变：hot-cold mixture 以及 spot 省钱 + burstable 作 backup recovery。
