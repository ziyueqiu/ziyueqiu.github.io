---
layout:     post
title:      Understand-Timeout (IC2E 18) 论文阅读
date:       2023-01-15
tags:
    - find-bug
    - timeout
categories: paperreading
comments: true
---

[Understanding Real-World Timeout Problems in Cloud Server Systems](https://tingdai.github.io/files/understanding_IC2E18.pdf)

我是 find-bug 小白，所以会记录很多基础信息：）

## Abstract

Timeouts are commonly used to handle unexpected failures in distributed systems.

- study 11 commonly used cloud server systems (e.g., Hadoop, HDSF, Spark, Cassandra, etc.)
- root causes of timeout problems include misused timeout, missing timeout, improper timeout handling, unnecessary timeout, and clock drifting

## Introduction

Timeout mechanism can be used in both intra-node and inter-node communication failover. For example, when a component C1 sends a request to another component C2, C1 sets a timer timeout value and waits for the response from C2 until the timer expires. In case C2 fails or a message loss occurs, C1 can break out of the waiting state triggered by the timeout event and take proper actions (e.g., retrying or skipping) accordingly.

一个例子：In 2015, Amazon DynamoDB service was down for five hours. The service outage is caused by a timeout bug in the metadata server. When the metadata server was already overloaded, the new requests from storage servers to the metadata server failed due to timeout. Storage servers kept retrying, causing further failures and retries, creating a cascading failure.

{% include figure.html path="assets/img/fig/Understand-Timeout-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

- Timeout bug root causes: Our study shows that real-world timeout problems are caused by a variety of reasons including
  - 1) misused timeout value (47%) where a timeout variable is misconfigured, ignored or incorrectly reused;
  - 2) missing timeout checking (31%) where inter-component communication lacks timeout protection; 
  - 3) improper timeout handling where a timeout event is handled by inappropriate retries or aborts; 
  - 4) unnecessary timeout where a timeout is used for a function call which does not need timeout protection;
  - 5) clock drifting where timeout problems are caused by asynchronous clocks between distributed hosts.
- Timeout bug impact
- Timeout bug diagnosis: Our study shows that real-world timeout bugs are difficult to diagnose: 60% timeout bugs do not produce any error message and 12% timeout bugs produce misleading error messages.

## Methodology

## Root cause analysis

很多例子！非常有意思！

## 评论

well-written! 下一篇准备读 follow-up 的解决方案 :)