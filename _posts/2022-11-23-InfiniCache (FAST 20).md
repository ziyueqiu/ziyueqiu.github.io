---
layout:     post
title:      InfiniCache (FAST20) 论文阅读
date:       2022-11-23
tags:
    - Cloud
    - Cache
categories: paperreading
comments: true
---

[InfiniCache: Exploiting Ephemeral Serverless Functions to Build a Cost-Effective Memory Cache](https://www.usenix.org/conference/fast20/presentation/wang-ao)

非常有意思（虽然不一定实用），用 lambda function 作 object caching，在 light load 下省钱效果佳。

## Abstract

- exploit serverless functions' memory resources to enable elastic pay-per-use caching
- + erasure coding
- + intelligent billed duration control
- + data backup mechanism 

关注 large-object-only production workload

## Introduction

In-Memory Object Caches (IMOCs) are largely used only as a small cache for buffering **small-sized objects**.

Large objects (MBs-GBs) consume significant memory capacity and network bandwidth + cause cache churn with evictions of many small objects.

IBM Docker registry traces: large objects also have good locality.

Providers only charge tenants when a function is invoked. 这个 cache 如果被 access 的频率不高，那会有可能比按整个时间段存的 cache 更划算。

## Background and Motivation

large object caching 的研究比较少（相比 Redis / Memcached 这些）。

### Large Object Caching

- Extreme Variability in Object Size
- Tension between Small and Large Objects
  - large objects 加起来传输占比超过 95%，但 IOPS < 3500 GETs/hour
- Caching Large Object Matters
  - 37%-46% large objects are reused within 1 hour since last time they were accessed

### Building a Memory Cache on Cloud Functions: Opportunities and Challenges

opportunities:

- pay-per-use pricing
- short-term caching

challenges of lambda function:

- a limited CPU and memory capacity
- run at most 900sec
- only allow outbound TCP network connections (not stateful)
- lack of quality-of-service (QoS) control

## InfiniCache Design

略，参考以下即可：

- [InfiniCache: Distributed Cache on Top of AWS Lambda (paper review)](https://mikhail.io/2020/03/infinicache-distributed-cache-on-aws-lambda/) 写得非常全面！并且比论文更方便读者！