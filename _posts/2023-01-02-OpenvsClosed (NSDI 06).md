---
layout:     post
title:      OpenvsClosed (NSDI 06) 论文阅读
date:       2023-01-02
tags:
    - Systems
categories: paperreading
comments: true
---

[Open Versus Closed: A Cautionary Tale](https://www.usenix.org/legacy/event/nsdi06/tech/full_papers/schroeder/schroeder.pdf)

因为我上课已经学了基础知识，可能会略过一些细节。

## Abstract

Using a combination of implementation and simulation experiments, we illustrate that there is a vast difference in behavior between open and closed models in real-world settings. We synthesize these differences into eight simple guiding principles, which serve three purposes. First, the principles specify how scheduling policies are impacted by closed and open models, and explain the differences in user level performance. Second, the principles motivate the use of partly open system models, whose behavior we show to lie between that of closed and open models. Finally, the principles provide guidelines to system designers for determining which system model is most appropriate for a given workload.

## Introduction

{% include figure.html path="assets/img/fig/OpenvsClosed-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

{% include figure.html path="assets/img/fig/OpenvsClosed-fig2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

## Closed, open, and partly-open systems

基础知识略

## Comparison methodology

涉及到的 policy：

- FCFS (First-Come-First-Served) Jobs are processed in the same order as they arrive.
- PS (Processor-Sharing) The server is shared evenly among all jobs in the system.
- PESJF (Preemptive-Expected-Shortest-Job-First) The job with the smallest expected duration (size) is given preemptive priority.
- SRPT (Shortest-Remaining-Processing-Time-First): At every moment the request with the smallest remaining processing requirement is given priority.
- PELJF (Preemptive-Expected-Longest-Job-First) The job with the longest expected size is given preemptive priority. PELJF is an example of a policy that performs badly and is included to understand thefull range of possible response times.

## Real-world case studies

{% include figure.html path="assets/img/fig/OpenvsClosed-fig3.png" class="img-fluid rounded z-depth-1" zoomable=true %}

注意对于 open system，(b) 多种 policy 区别不大是因为 E[S] 波动不大，当然，相比 closed system 整体差别还是很大的。

从阐述的细节来看，每个实验都修改了代码来实现 default 之外的 policy，非常扎实！

另外，partly-open system 怎么 simulate 也比较有意思。

## Open versus closed systems

- Principle (i): For a given load, mean response times are significantly lower in closed systems than in open systems.
- Principle (ii): As the MPL grows, closed systems become open, but convergence is slow for practical purposes.
- Principle (iii): While variability has a large effect in open systems, the effect is much smaller in closed systems.

{% include figure.html path="assets/img/fig/OpenvsClosed-fig4.png" class="img-fluid rounded z-depth-1" zoomable=true %}

- Principle (iv): While open systems benefit significantly from scheduling with respect to response time, closed systems improve much less.
- Principle (v): Scheduling only significantly improves response time in closed systems under very specific parameter settings: moderate load (think times) and high MPL.
- Principle (vi): Scheduling can limit the effect of variability in both open and closed systems.

## Partly-open systems

- Principle (vii): A partly-open system behaves similarly to an open system when the expected number of requests per session is small (≤ 5 as a rule-of-thumb) and similarly to a closed system when the expected number of requests per session is large (≥ 10 as a rule-of-thumb).
- Principle (viii): In a partly-open system, think time has little effect on mean response time.
    - To change the load, we must adjust either the number of requests per session or the arrival rate. The only effect of think time is to add small correlations into the arrival stream.

## 评论

好像和上课重合度挺大的，在舒适区内：）