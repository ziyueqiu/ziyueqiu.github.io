---
layout:     post
title:      InftyDedup (FAST 23) 论文阅读
date:       2023-03-01
tags:
    - Cloud
    - Deduplication
categories: paperreading
comments: true
---

[InftyDedup: Scalable and Cost-Effective Cloud Tiering with Deduplication](https://www.usenix.org/conference/fast23/presentation/kotlarska)

这方面了解不多，所以会记录一些知识：）

## Abstract

- maximize scalability by utilizing cloud services not only for storage but also for computation
- save cost by selecting between hot and cold cloud storage based on the characteristics of each data chunk

## Introduction

- Deduplication between different local tier systems is not supported for data stored in the cloud.
- Deduplication typically entails chunking data collections into smaller pieces that may be referenced multiple times, thereby possibly having different access patterns. This calls for finer-grained and more automated approaches to storage type selection.

Conclusion:

- Like the existing tiering-to-cloud backup solutions, InftyDedup moves selected data from a local-tier system to the cloud, based on customer-specific backup policies.
- Rather than relying on deduplication methods of on-premise solutions, InftyDedup deduplicates data using the cloud infrastructure. (也就是说连 dedup 计算本身都可以做到不依赖本身的算力，读到现在我猜是也可以租 VM。) This is done periodically in batches before actually transferring data to the cloud, which, among others, enables the dynamic allocation of cloud resources. 

## Background

### Deduplication Storage

Firstly, the data stream is chunked into small immutable blocks of size from 2 KB to 128 KB. Secondly, each block receives a fingerprint, for instance, by computing the SHA-256 hash of the block’s data. Finally, the fingerprint is compared with other fingerprints in the system, and if the fingerprint is unique, the block’s data is written.

The deduplicated blocks are typically organized in a directed acyclic graph. Each file has its root block corresponding to a vertex that references other blocks.

### Lifecycle of Backups

两个约束：
- the data should be available quickly in case of a disaster
  - e.g. Recovery Point Objectives of seconds and Recovery Time Objectives of minutes
- older versions of backups need to be stored for weeks, months, or even years
  - backups are often moved to cheaper storage after a specific time

### Cloud Storage

{% include figure.html path="assets/img/fig/InftyDedup-fig1.png" class="img-fluid rounded z-depth-1" zoomable=false %}

## InftyDedup Architecture

The cloud tier stores deduplicated data with necessary persistent metadata, and occasionally executes highly optimized batch algorithms.

{% include figure.html path="assets/img/fig/InftyDedup-fig2.png" class="img-fluid rounded z-depth-1" zoomable=false %}

### Cloud Cost Considerations

暂略

## 评论

- 我们能认为用户级别的 dedup 是在榨取 cloud providers 的利润吗？因为他们肯定也会做 dedup。
- 总的来说就是几方面的 tradeoff：deleted blocks 要存多久再做 GC？removal cost vs storage cost，根据 backup 身上的 expiration date 计算；要放在多便宜的 layer，根据各种 cost，比如 data movement / restore frequency 等等计算。对于 backup，这一切格外地好计算。 