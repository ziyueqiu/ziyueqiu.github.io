---
layout:     post
title:      RocksDBWorkload (FAST 20) 论文阅读
date:       2023-06-20
tags:
    - Modeling
categories: paperreading
comments: true
---

[Characterizing, Modeling, and Benchmarking RocksDB Key-Value Workloads at Facebook](https://www.usenix.org/system/files/fast20-cao_zhichao.pdf)

我比较关心 benchmarking，而且本文感觉有追随 Sigmetrics 12 的工作，我挑不同的地方记录。而且已经有一些 blog 写这篇文章，我就尽量精（tou）简（lan）了。

## Introduction

- the accesses in UDB explicitly exhibit a diurnal pattern -- How to model diurnal patterns?
- The whole keyspace is partitioned into small key-ranges, and we model the hotness of these small key-ranges.
- YCSB causes at least 500% more read-bytes and delivers only 17% of the cache hits in RocksDB compared with realworld workloads. New benchmark has only 43% more read-bytes and achieve about 77% of the cache hits in RocksDB, and thus are much closer to real-world workloads.

## Modeling and Benchmarking

We first calculate the Pearson correlation coefficients between any two selected variables to ensure that these variables have very low correlations. In this way, each variable can be modeled separately. Then, we fit the collected workloads to different statistical models to
find out which one has the lowest fitting error, which is more accurate than always fitting different workloads to the same model (like Zipfian). The proposed benchmark can then generate KV queries based on these probability models.

QPS (Queries Per Second) of some CFs in UDB (social network SQL) have strong diurnal patterns

### Key-Space and Temporal Patterns

We sort all the existing keys **in the same order as they are stored in RocksDB** and plot out the access count of each KV-pair, which is called the heat-map of the whole key-space.

They mentioned: The hotness of these key-ranges is closely related to cache efficiency and generated storage I/Os. The better a key-space locality is, the higher the RocksDB block cache hit ratio will be.

### Key-Range Based Modeling

We first fit the distributions of key sizes, value sizes, and QPS to different mathematical models (e.g., Power, Exponential, Polynomial, Webull, Pareto, and Sine) and select the model that has the minimal fit standard error (FSE). This is also called the root mean squared error.

QPS can be better fit to **Cosine or Sine** in a 24-hour period with very small amplitude

Details:

Based on the KV-pair access counts and their sequence in the whole key-space, the average accesses per KV-pair of each key-range is calculated and fit to the distribution model (e.g., power distribution). This way, when one query is generated, we can calculate the probability of each key-range
responding to this query. Inside each key range, we let the KV-pair access count distribution follow the distribution of the whole key-space. This ensures that the distribution of the overall KV-pair access counts satisfies that of a real-world workload. Also, we make sure that **hot KV-pairs are allocated closely together**. Hot and cold key-ranges can be randomly assigned to the whole key-space, since the locations of keyranges have low influence on the workload locality.

我的理解是：

- 首先对于多个 key-range，各自有不同的热度，他们之间的位置的相对关系不重要，生成的时候先根据概率选到某个 key-range；
- 对于每个 key range，不区分他们的 distribution，都使用整体的 distribution（e.g. power distribution）来区分冷热块；
- 需要注意 hot KV-pairs 还需要物理上接近

目前为止我的疑问有：

- 怎么切分 key-range？太细或者太粗粒度效果都不好
- 怎么让 hot KV-pairs 物理上接近？
- 如何处理 request type generation？Put, Get, Delete, etc；我认为不同类型的 request，distribution 也不同
