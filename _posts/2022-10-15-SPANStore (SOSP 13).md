---
layout:     post
title:      SPANStore (SOSP 13) 论文阅读 - span multiple cloud for cost-saving and low latency
date:       2022-10-15
tags:
    - Cloud
categories: paperreading
comments: true
---

[SPANStore: cost-effective geo-replicated storage spanning multiple cloud services](https://dl.acm.org/doi/10.1145/2517349.2522730)

这篇文章的核心是通过 span multiple cloud provides 获得两个好处：1) increase **geographical density** of data centers; 2) minimze cost by **exploiting pricing discrepancies** across providers

补充：选择空间从几个到了20个这种感觉。

## Abstract

从这里看到的几个要点：

- storage services in geographically distributed data centers
- minimize cost（和纯 performance 视角不同）
- trade off replication with 'higher storage' and **data propagation costs** (i.e. transfer cost), 还需要满足 fault tolerance and consistency requirements
-  also optimize 'compute resources' used on tasks such as two-phase locking and data propagation

到这里有几个疑问在我心里：

- ~~geographically distributed data centers 有哪些在意的维度？目前看来至少有 cost, latency, fault tolerance, and consistency~~ Yes, in this paper
- 什么是 higher storage
- ~~什么是 'compute resources' used on tasks such as two-phase locking and data propagation~~

看到目前觉得这篇文章考虑的视角并不狭窄（我不敢确定是不是足够全面），适合刚入门（比如我）学习。我是冲着 'exploiting pricing discrepancies' 和 'data propagation costs' 来的 :)

## Introduction

首先记录一下这里提到的常见云服务（Amazon S3, Google Cloud Storage (GCS), Microsoft Azure），之后读近年的文章可能会看到不同的 set :)

通过 PUT 和 GET 处理数据（PS: 据说近几年也有加入多一些基础 Operation 的趋势）。为了保证比较好的延迟，application 需要提供用户数据时最好从距离用户近的 data center 取。

challenges (complexity):

- 云服务商把操作 replication 的责任交给 application，他们只管给 'an isolated pool of storage'（这里默认需要 'replication' 是因为数据如果不在多个服务器上都有备份，谈何从最近的里面取呢？）
- 如果干脆不管了，就每个 data center 都有一份全部的数据呢？性能（latency for this paper）确实没毛病，但是太贵了，很多 application 还是希望刚好满足性能要求的情况下最便宜的方案；更何况数据和 client 都有自己的特征，全部来一份完整的显然太多冗余/浪费
- 云服务商把责任交给 application 是合理的，因为服务商的 level 很可能会缺失 hint / semantics / knowledge 等

Mark: SPANStore = Storage Provider Aggregating Networked Store

Three key principles:

- span cloud providers（地理上看可选的 data center 更多了，还可以根据价格差异选最便宜的）；
- 对于 Key-Value Store，哪些 object 要放在哪，以及从哪传输出来，都是责任；
  - 影响因素：workload, latency requirement, fault tolerance, consistency requirement, pricing model（和之前完全对应）
- 第三点的细节**没看懂**：To keep costs low, we ensure that all data is largely exchanged directly between application virtual machines (VMs) and the storage services that SPANStore builds upon; VMs provisioned by SPANStore itself rather than by application provider are predominantly involved only in metadata operations.

~~Mark: 号称有两个 killer application~~ 最后有讲到

Question:

- 这些云服务的特点和差异都是什么呢？
- ~~后面会有完整的云服务商价格差异表吗？~~ 现在情况如何了呢？
- ~~和 single cloud 的核心区别在哪？single cloud的这种系统同样考虑 'workload, latency requirement, fault tolerance, consistency requirement'，这篇文章的贡献只是多了 ‘pricing model’？~~ Figure 9

lecture 里面放了一张图：

{% include figure.html path="assets/img/fig/SPANStore-fig3.png" class="img-fluid rounded z-depth-1" zoomable=true %}

关于云服务商的背景，推荐一波 [From Cloud Computing to Sky Computing](https://sigops.org/s/conferences/hotos/2021/papers/hotos21-s02-stoica.pdf) \[HotOS 21\]，有讨论 cloud 的一些特点。

## Problem formulation

跨云服务商的讨论，只考虑 **storage** 不考虑 **computing**（since cloud computing platforms vary in the abstractions）

目标：

- cost (both storage cost and compute cost)
- latency
- fault tolerance
- consistency

Challenges（从写作角度看写得挺 convincing）：

- Inter-dependencies between goals
- Dependence on workload
  - 一个有趣的例子：to reduce network transfer costs, it is more cost-effective to replicate the object more (less) if the workload is dominated by GETs (PUTs).
- Multi-dimensional pricing
  - 维度很多：存储数据的量，PUT & GET 的价格，transfer data 的量多不多（以及传到多远）
  - Note: transfer cost 和 PUT/GET cost 不同

## Why multi-cloud

Lower latencies & Lower cost

从写论文角度看，这个图画的不错

{% include figure.html path="assets/img/fig/SPANStore-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Question: **为什么 a & b 图（c & d）的红色长得不一样？**

## Overview
{% include figure.html path="assets/img/fig/SPANStore-fig2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

Mark: PMan = Placement Manager

这图信息量挺大。

这里提到一个细节：

- Divide time into fixed-duration **epochs**; an epoch lasts one hour in current implementation

这是系统文章常见的操作——总是存在“magic number” :)

## Determining replication policies

### Inputs and output

首先，这里终于解释了 consistency：

- To capture consistency needs, we ask the application developer to choose between **strong** and **eventual** consistency.
- In the strong consistency case, we provide **linearizability**, i.e., all PUTs for a particular object are ordered and any GET returns the data written by the last committed PUT for the object.
- In contrast, if an application can make do with eventual consistency, SPANStore can satisfy lower latency SLOs. Our algorithms for the eventual consistency scenario are extensible to other consistency models such as causal consistency [26] by augmenting data transfers with additional metadata.

留下关于 eventual consistency 的疑问，可能需要读 citation（[26] Don’t settle for eventual: Scalable causal consistency for wide-area storage with COPS [SOSP 11]）

第二，提出了 *Access set* 概念: 每个 object 会被哪些 data center 访问（无信息也可以设置为 all），但是本文假设是有这个信息的。

Questions:

- ~~感觉 objects grouped 之后 metadata 还是太多了吧？还是说如果有 N 个 data center，有 2^N 种分类而不是每个 object 有 access set list？~~ 是后者。

第三，利用 diurnal and **weekly** patterns [11, 17] 

**stationary**: 关于 weekly，分析每个**小时**（也是每个epoch）与一周前的那个小时的 relative difference，发现虽然单用户指标不是很好看，但是当 "consider those users whose timelines have their access set as all EC2 data centers in US"**（没懂这个 category 是怎么分的）**，relative difference 恒小于 50%。

intuitively: 某个用户不一定稳定在周一早上 7 点触发 GET request，但是用户群整体可能有相对稳定（50% relative difference in this paper）的访问量。

好处：enable more accurate prediction based on historical workload measurements

([11] Volley: Automated data placement for geo-distributed cloud services [NSDI 11], and [17] Characterizing, modeling, and generating workload spikes for stateful services [SoCC 10])

第四，讲到了 replication policy：1）为每个 access set（会被相同的 data center 访问的一群 object）决定哪些 data center 拥有它们；2）为 access set 里的那些 data center 决定从哪里读写数据。

Questions:

- ~~access set 顶多是一种访问权限的组，为什么有必要/有好处作为一个组来考虑 replication 呢？~~ 我觉得是为了 simplicity，不然复杂度更高。

### Eventual consistency

不让所有 data center 拥有全部的数据的原因：1）storage cost 太高；2）PUT 需要对所有备份做一遍，PUT request cost 太高。

Solution: determine the replication policy for a given access set AS as a mixed integer program.（整型规划，这里要解的是0/1规划：whether a data center is GET/PUT replica）

Appendix A:

- 因为 fault tolerance = f，所以要有 f+1 个 replica（这里是说有 f+1 个备份就可以支持 f 个 data center 里内容其实不是 up-to-date 的，倒不是说 corruption）；
- 因为 PUT/GET replica set 不需要重合也确实不重合，所以它们的并集内的 data center 都有全部的 object；
- 对于 eventual consistency，PUT 只需要同步到某一个 replica，之后都异步做即可；
- 有时候多几跳（relay via other data centers）反而比直接传输能节省 networking cost。

Question:

- Assume PUT 可以多几跳，但是 GET 是直接从 replica 获得，为什么不考虑 relay 呢？有意思的 Related work: [Skyplane: Optimizing Transfer Cost and Throughput Using Cloud-Aware Overlays](https://arxiv.org/pdf/2210.07259.pdf)

### Strong consistency

rely on quorum consistency

**暂略（包括了 Appendix B）**

## SPANStore dynamics

### Metadata

"At any data center A, SPANStore stores an (a) in-memory version mapping for objects stored at A. If the application is deployed at A, SPANStore also stores (b) the access set mapping for objects whose access set includes A, and (c) **replication policy versions for different epochs**."

"when serving the **first operation** for an object in a particular epoch, SPANStore needs to account for both the replication policy currently in use for that object and the new replication policy computed by PMan for the current epoch."

### Serving PUTs and GETs

Eventual consistency: 获得了至少一个replica的回复（PUT/GET）

Strong consistency: 2PL (PUT)

- 需要等所有的replica

Modified 2PL protocol: 1) PUT，不管锁，直接写，留下版本号；2) GET，读到最近版本号到即可

### Fault tolerance

Eventual consistency: 利用存储层次的保障，应该不会丢数据吧（=啥也没有）

Strong consistency: "quorum sets"

### Handling workload changes

如何识别 replication policy 切换后的 **first operation**？version mismatch！

这里讨论了识别出来分别是 GET 和 PUT 要怎么处理（略，因为 intuitive）。

Question:

- **但是 policy 的变化是某个 Access Set 在 Data center 之间整体迁移的大动作，这里凭什么只讨论 first operation 呢？**


## Implementation

On Amazon S3, Microsoft Azure, and Google Cloud Storage

补充一点我查到的背景知识：

- **XML-RPC** is a remote procedure call (RPC) protocol which uses XML to encode its calls and HTTP as a transport mechanism.
- **The IBM ILOG CPLEX Optimizer** solves integer programming problems, very large linear programming problems using either primal or dual variants of the simplex method or the barrier interior point method, convex and non-convex quadratic programming problems, and convex quadratically constrained problems (solved via second-order cone programming, or SOCP).

一些关于具体实现的细节：

- "The client library exports two methods: GET(key) and PUT(key, value, [access set])"
- XMLRPC exports LOCK(key) RPC to acquire object-specific locks; RELAY(key, data, dst) to indirectly relay a PUT; receives replication policy updates fromm PMan

Question:

- ~~为什么 GET 和 PUT 的函数输入不一样？~~ 可能因为 PUT 有可能是新的
- "uses DNS to discover the local memcached cluster" 从何说起？

## Evaluation

- cost savings
- cost-optimality of replication policies
- cost necessary for increased fault-tolerance
- scalability of PMan

Note: application on EC2's data centers; SPANStore's storage services offered by S3, Azure, GCS.

### Cost savings

关于 SLO 的设定，有意思的细节是：for GET，250 ms is minimum SLO with "single replica for every object"; for PUT, 830 ms is minimum SLO with "replication everywhere".

Figure 9 画的不好看清楚。

我没有细看每条线的对比，我感觉和 Single or Everywhere 比，赢了很合理；和 single-cloud 比，应该是赢在 data center 选择更多和价格差异上。

补充一下：这个工作不止是可以优化拓扑结构，也可以减少 replica 的绝对数量（选择多了，也许一个 DC 可以保证原先不得不用两个 DC 才能做到的 SLO）；cost 是 goal 的话，资源的节省有可能伴随 cost 产生。

### Impact of aggregation of objects

这里我觉得视角不够好。

论文在 Figure 12 给出了 per-object 预测效果远不如 aggregate workload 的效果好，但这只够论证选 per-object 预测不可行，不能说明 aggregate 可行。当然，本节标题也只是说了 aggregation 很有效果。

结合前面给过的一个 aggregate workload relative difference 50% 以内的数据，算是圆上了。

### Cost for fault tolerance

**略**

### Scalability of PlacementManager

"PMan needs to compute the replication policy for all access sets; there are 2^N access sets for an application deployed across N data centers."

CPU 算力只能支持 15 及以下数量的 data center，但实际上不需要每个 epoch 都重新计算（workload 变了再说）。

感觉确实不是很 scalable，虽然对当时的场景够用了。

## Case studies

killer application 来了！

- Retwis is a clone of the Twitter **social networking** service (w. eventual consistency)
- ShareJS is a collaborative document **editing** webservice (w. strong consistency)

这部分有点少。只展示了两个 application （两种 consistency requirement）下 SPANStore 都可以“正确地”满足 SLO，但没有说省到钱没有（或者原先是怎么做的）。

## Related work

- Evaluating benefits of cloud deployments
  - 视角不够全面
- Using multiple cloud services
  - 此前只考虑了：availability, durability, vendor lock-in, performance, and consistency（没有 cost）
  - SPANStore "unify idea"
- Optimizing costs and scalable storage
  - 没能跨 data center 考虑问题
- Low-latency geo-replicated storage with improved consistency
  - 没有考虑 minimize cost

## 评价

- 没有开源
- 虽然 related work 不够 convincing（比如我总觉得这种整型优化肯定有人做了的，但是却没有比较到），但是如果我是 PC，我会觉得这个工作还是填补了空缺的、值得 accept 的
- 整篇看下来不细想的话没啥问题，但是一细想就发现很多细节都不知道是如何做的，或者有一些假设是有可能不合适的。
- 不知道后续有没有足够有意思的 follow-up（比如应用在了哪里）。
- 我还看见[一篇介绍 SPANStore 的文章](http://dsrg.pdos.csail.mit.edu/2013/09/30/spanstore/)（来自 MIT）。一段有意思的评价：It seems that there is still a huge burden on developer to provide the correct inputs to PMan so that PMan can provide the best replication policy. This doesn’t seem to reduce the complexity involved. A lot of the paper relies on the objective function, and there are not many new distributed system concepts.