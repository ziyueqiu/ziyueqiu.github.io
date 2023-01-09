---
layout:     post
title:      SaveCash (HotCloud 12) 论文阅读
date:       2023-01-03
tags:
    - Cache
    - Cloud
categories: paperreading
comments: true
---

[Saving Cash by Using Less Cache](https://www.usenix.org/conference/hotcloud12/workshop-program/presentation/zhu)

非常干净简洁的 paper，和 [Adaptive Hashing](https://www.usenix.org/conference/icac13/technical-sessions/presentation/hwang) 那篇有点关联。

## Abstract

This paper makes the case that (i) scaling down the caching tier is viable with respect to performance, and (ii) the savings are potentially huge; e.g., a 4x drop in load can result in 90% savings in the caching tier size.

## Introduction

When the overall load drops, we can afford a higher fraction of requests going to the data tier; hence, we can tolerate a lower cache hit rate, and this lower cache hit rate translates to a vast reduction in the amount of data cached.

{% include figure.html path="assets/img/fig/SaveCash-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

虽然 Figure 2 取的例子看到 Zipf 有可能可以榨出很多 cache size，但是注意这是在比较高的命中率往下降的情况下才成立。

## Experimental setup

## How much cache can we save?

## Managing the caching tier

scaling down: To avoid spike in response times, we propose **gradually migrating** the most popular data off of the instance to be removed. Conceptually, we treat the instance to be removed as a **second level cache** for some period of time. If we miss in the rest of the cache, then we query this instance before going to the database. This will naturally migrate the popular items from this “retiring” instance to the rest of the caching tier.

## 评论

不要害怕 scaling down 和 scaling up！