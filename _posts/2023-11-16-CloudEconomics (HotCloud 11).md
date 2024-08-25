---
layout:     post
title:      To Move or Not to Move\: The Economics of Cloud Computing (HotCloud 11) 论文阅读
date:       2023-11-16
tags:
    - cloud
categories: paperreading
comments: true
---

[To Move or Not to Move: The Economics of Cloud Computing](https://www.usenix.org/legacy/event/hotcloud11/tech/final_files/Tak.pdf)

本文在 2011 年尝试做出了对往后十年的预测，有一些现在看来并不准确，但有一些地方作为早期的观察是很敏锐的。我抱着敬畏之心记录至今仍有借鉴意义的点。作为一个既不太懂 cloud，也不太懂 economy 的人，欢迎评论区纠正！

## Introduction & Background

Cloud-based hosting promises several advantages:

- Ease-of-management: since the cloud provider assumes management-related responsibilities,
the customer is relieved of this burden and can focus on its core expertise.
- Cap-ex savings: it eliminates the need for purchasing infrastructure; this may translate
into lowering the business entry barrier.
- Op-ex reduction: elimination of the need to pay for salaries, utility electricity bills, real-estate rents/mortgages, etc. One oft-touted aspect of Op-ex savings concerns the ability of customer’s Op-ex to closely match its evolving resource needs (via usage-based charging) as opposed to depending on its worst-case needs.

An interesting comment: using the cloud need not preclude a continued use of in-house infrastructure. The most cost-effective approach for an organization might, in fact, involve a combination of cloud and in-house resources rather than choosing one over the
other.

Two types of partitioning:

- Vertical partitioning splits an application into two subsets (not necessarily mutually exclusive) of components - one is hosted in-house and the other migrated to the cloud
- Horizontal partitioning replicates some components of the application (or the entire application) on the cloud along with suitable workload distribution mechanisms. Such partitioning is already being used as a way to handle unexpected traffic bursts by some businesses.

## Key Results

### Workload Intensity

In-house provisioning is cost-effective for medium to large workloads, whereas cloud-based options suit small workloads. For small workloads, the servers procured for
in-house provisioning end up having significantly more capacity than needed (and they remain under-utilized) since they are the lowest granularity servers available in market today. On the other hand, cloud can offer instances matching the small workload needs (due to the statistical multiplexing and virtualization it employs).

### Workload Growth

This paper assumes "hardware capacity growing according to Moore’s Law, unless the workload growth matches or exceeds this rate, the number of servers required in-house will actually shrink each year. However, things evolve differently with cloud-based options. The computing power as well as price of a cloud instance are intentionally engineered to be at a certain level (via virtualization and statistical multiplexing) even though cloud providers may upgrade their hardware regularly (just as in-house). E.g., since
the start of EC2 in 2006, the computing power/memory per instance has remained unchanged while there has been only one occasion of instance price reduction. In other words, while in-house hosting enjoys improvement in performance/$ with time, trends over the last 5 years suggest that **the performance/$ offered by the cloud has remained unchanged**. Even if we assume the performance/$ offered by the cloud improves with time (say, an instance of given capacity becomes cheaper over time), cloud-based provisioning still remains expensive in the long run since data capacity and transfer costs contribute to the costs more significantly than in-house."

We don't believe this is valid: at the beginning, AWS provided c4.xlarge and its prices may actually haven't changed for the last decade. But after some period of time, they started to provide upgraded VM, c5.xlarge, which is even cheaper than c4.xlarge with better performance. Similar to what's happening in in-house case.

### Data Transfer

Data transfer is a significant contributor to the costs of cloud-based hosting. This suggests that vertical partitioning choices may not be appealing for applications that
exchange data with the external world.

### Workload Variance and Cloud Elasticity

Horizontal partitioning can be effectively used to eliminate the cost increase from provisioning for the peak.

## 评论

简单来说，本文的一个重要假设 - cloud 的性价比会越来越差 - 严重影响了大部分的计算结果，而我们不太认同。但其他几个比较独立的观察 - data transfer 很贵、horizontal partitioning 天然地适配 burst load to cloud 的情况 - 应该是蛮先锋的断言。