---
layout:     post
title:      MemcachedWorkload (Sigmetrics 12) 论文阅读
date:       2023-06-19
tags:
    - modeling
    - cache
categories: paperreading
comments: true
---

[Workload Analysis of a Large-Scale Key-Value Store](https://ranger.uta.edu/~sjiang/pubs/papers/atikoglu12-memcached.pdf)

本文我比较关心 workload generation，其他部分我会挑值得记录的写一写。

## Overview

- analyze five workloads from Facebook’s Memcached deployment
- over 284 billion requests over a period of 58 sample days
- analyze request composition, size and rate, cache efficacy, and temporal patterns
- propose a Model to generate synthetic workloads

## Statistical Modeling

Use ETC trace, obviously not include all the nuances of the trace such as its bursty nature or the inclusion of one-off events.

- Emulate three properties: key size, value size, and inter-arrival rate['the time gap in microseconds between consecutive received requests']
- Use Pearson correlation coefficient, find these three are independent

{% include figure.html path="assets/img/fig/FacebookWorkload-fig1.png" class="img-fluid rounded z-depth-1" zoomable=false %}

- Fit distributions (such as Weibull, Gamma, Extreme Value, Normal, etc.) to each data set, and choose the one that minimizes the Kolmogorov-Smirnov distance
- Diurnal: divide raw data into 24 hourly bins and model each

## 评论

本文虽然号称是对 memcached 这个 caching engine 做的 workload 分析，实际上完全不能用来生成 caching workloads，因为甚至没有 differentiate different objects，那就是 uniform distribution，那就应该做 tiering 或者直接放一些数据再也不动了。
