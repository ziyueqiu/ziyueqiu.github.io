---
layout:     post
title:      MetastableFailure (OSDI 22) 论文阅读
date:       2022-12-31
tags:
    - Distributed Systems
categories: paperreading
comments: true
---

[Metastable Failures in the Wild](https://www.usenix.org/conference/osdi22/presentation/huang-lexiang)

作为分布式系统的小白，本文会记录下很多基础知识 :(

## Abstract

本文关注 deep understanding of metastable failures，并没有涉及到解决它。

** hyperscalers: those massive companies like Google, Facebook, and Amazon that are making efforts to not only dominate the public cloud and cloud services industries, but to expand their business into numerous related verticals, as well

## Introduction

fail-stop failures: In this type of failure, the server only exhibits crash failures, but at the same time, we can assume that any correct server in the system can detect that this particular server has failed.

state definition:

- metastable failure state: the state of a permanent overload with an ultra-low goodput (throughput of useful work)
- stable state: the state when a system experiences a low enough load than it can successfully recover from temporary overloads
- vulnerable state: the state when a system experiences a high load, but it can successfully handle that load in the absence of temporary overloads

The distinguishing characteristic of a metastable failure is that the sustaining effect keeps the system in the metastable failure state even after the trigger is removed.

本文的贡献主要是：

- A study of metastable failures in the wild that confirms metastable failures are universally observed and comprise a substantial fraction of the most severe outages.
- An improved model that categorizes two types of triggers and two types of amplification mechanisms, which better explains how metastable failures happen.
- An insider view at Twitter of a new type of metastable failure where garbage collection acts as an amplification mechanism.
- [Three example](https://github.com/lexiangh/Metastability) applications on which metastable failures are experimentally reproduced, which helps researchers propose and test solutions to metastable failures.

## Metastability in the Wild

{% include figure.html path="assets/img/fig/MetastableFailure-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

While triggers initiate the failure, the sustaining effect mechanisms prevent the system from recovering. We observed a variety of different sustaining effects, such as load increase due to retries, expensive error handling, lock contention, or performance degradation due to leader election churn. By far, the most common sustaining effect is due to the retry policy, affecting more than 50% of the studied incidents.

## Metastability Framework

这个 framework 为了照顾 system 人 做到了非常 intuitive，我甚至感觉有点过于简单了。

{% include figure.html path="assets/img/fig/MetastableFailure-fig2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

- Definition 1 (Trigger). A trigger represents the total effect from one or more of the following events:
  - A load-spike trigger is an event that increases the load on the system by some maximum magnitude **mtrigL**
  - A capacity-decreasing trigger is an event that decreases the system capacity by some maximum magnitude **mtrigC**
- Definition 2 (Overloading trigger condition). If mtrigL + mtrigC ≥ Cnorm − Lnorm, then the trigger(s) can overload the system.
- Theorem 1 (Overloading trigger). If the system does not have an overloading trigger condition, then it will never have a metastable failure.
- Definition 3 (Sustaining effect). A sustaining effect is a feedback loop that keeps the system in an overloaded state such that Lsys(t) ≥ Csys(t) even after the trigger is removed.
- Definition 4 (Metastable amplification). A metastable amplification exacerbates the system’s overload until it reaches a maximum overload limit. 
- Theorem 2 (Stable region). Define $$C_{stable}=\frac{C_{norm}}{w_L^**w_C^*}$$. If
Lnorm < Cstable, then the system will never have a metastable failure.
- Theorem 3 (Degrees of vulnerability). If the metastable amplification during the trigger overloading duration ∆$$t_{trig}$$ is small enough relative to the system headroom (i.e., wL(∆t$$t_{trig}$$) * wC(∆t$$t_{trig}$$) < $$\frac{C_norm}{L_norm}$$ ), then the system will never have a metastable failure.
- Theorem 4 (Metastable failure boundary). If the metastable amplification causes the system overload to exceed the triggers’ effects (i.e., Lsys(t) − Csys(t) ≥ αL(t) * mtrigL + αC(t) * mtrigC), then the system is in a metastable failure state.

Intuitively, the upper bounds allow us to reason about vulnerability and when a system enters a metastable failure state. For instance, a policy with at most two retries will not amplify the work more than three times, **while the policy with no cap effectively leaves the system with no stable region**.

{% include figure.html path="assets/img/fig/MetastableFailure-fig3.png" class="img-fluid rounded z-depth-1" zoomable=true %}

**Timeout**: A short request timeout is good for latency when retrying due to a small transient issue. However, it can hurt the system’s ability to handle larger problems by quickly starting the workload amplification. A longer timeout would have both reduced the load and allowed for a longer wait, potentially delaying workload amplification.

## Metastability at Twitter

We identify a sustaining loop where high queueing increases memory pressure and mark-and-sweep processing during GC, causing job slowdowns and thus higher queueing.

Engineers take different approaches to mitigate/fix this issue. For example, (i) observing unusually high latency spikes in backend services resulted in work to improve their performance to lower queue lengths, (ii) observing higher GC duration than normal resulted in adjusting the JVM memory configuration (e.g., increasing max heap size) to tweak GC behavior, and (iii) observing high resource utilization (e.g., CPU) resulted in adding more servers to lower per-server load. These approaches decrease system vulnerabilities and make it more robust to the trigger at the magnitude of the peak load test level.

## Replicating Metastability

三个例子：1) 前面提到的 GC; 2) retries; 3) look-aside cache

第三个比较有意思的是：Actually, caching systems by nature are self-healing, and we would expect the system to eventually recover if there’s a non-zero chance that a request would successfully add data to the cache.

There is a trade-off with setting the request timeout—a higher timeout decreases the vulnerability, but it takes longer to detect failed requests, whereas a lower timeout can quickly detect issues, but increases the metastable vulnerability.

## 评论

问题：

- 一直不太明白为什么 retry 会很大程度地影响 load，那些已经 timeout 的请求难道不是应该很快被发现不需要花时间处理、直接丢弃掉？