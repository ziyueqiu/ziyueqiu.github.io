---
layout:     post
title:      Glider (MICRO 19) 论文阅读
date:       2022-12-23
tags:
    - Cache
    - ML
categories: paperreading
comments: true
---

[Applying Deep Learning to the Cache Replacement Problem](https://www.cs.utexas.edu/users/lin/papers/micro19c.pdf)

（号称）第一篇 Deep Learning + Cache Replacement 走进现实

注意：本人 ML 白痴，本文可能会记下很多白痴知识 :(

## Abstract

- a powerful LSTM learning model can in an offline setting provide better accuracy than current hardware predictors
- a simple online model that matches the offline model’s accuracy with orders of magnitude lower cost（Q:**相对** cost 降低也不一定能放在硬件？）

Questions:

- 啥是 2nd Cache Replacement Championship
- 如何解释多核的效果比单核明显？

## Introduction

deep learning 的好处是：The use of multiple layers is significant because it provides the ability to learn non-linear relationships at multiple levels of abstraction.

当前 cpu cache + learning 的现状是 perception 或者 table，前者即 single layer of neural network

Challenge - "deep learning models are ill suited for use in hardware predictors"

- require enormous resources to train
  - offline training is much less effective for hardware predictors because (1) computer programs exhibit time-varying phase behavior, so prediction targets change as the program executes, and (2) the behavior of one program can differ wildly from that of another.
- (even with compression) too large to be implemented on a chip to make predictions
- slow, typically taking several milliseconds to produce a prediction, while hardware predictors typically need a prediction within nanoseconds.

Glider:

- design a powerful, unconstrained deep RNN model that is trained offline for individual programs
  - "sequence labeling problem": assign each element in a sequence of memory accesses a binary label that indicates whether the accessed data should be cached or not
  - use Belady's MIN algorithm to provide oracle labels for training data
  - -> **attention-based** LSTM (Long Short Term Memory, a special variant of recurrent neural networks that can effectively learn long-term dependences in sequential data) as model (82.6% vs. 72.2% for prior art)
- An analysis of the attention weights inside the LSTM then reveals several **insights**:
  - the LSTM benefits when given as input a long history of past load instructions, so we conclude that optimal caching decisions depend on the program’s control-flow history and that a long control-flow history is beneficial.
  - the optimal decisions depend primarily on **the presence of a few elements** (???) in the control-flow history, not the full ordered sequence.
- use these insights to design a simple online model, support vector machine (SVM)
  - trained online in hardware
  - match the LSTM’s offline accuracy with significantly less overhead
  - with our hand-crafted feature, an online SVM is equivalent to a perceptron, which has been used in commercial branch predictors.

Questions:

- "offline model should be able to reason about **time-ordered events**" (???)

## Related Work

### Cache Replacement

- Heuristic-Driven Solutions. They are customized for a limited class of known cache access patterns.
- Learning-Based Solutions: to predict whether an incoming line is cache-friendly or cache-averse
  - Hawkeye learns from the optimal solution for past accesses. By providing oracle labels for past accesses, Hawkeye phrases cache replacement as a supervised learning problem. Glider builds on Hawkeye, but we **use deep learning to design a better predictor** than Hawkeye’s PC-based predictor.
- Machine Learning-Based Solutions. Glider uses an **unordered history** of **unique PCs**.
  - First, because it does not include duplicate occurrences of the same PC, Glider uses an effectively longer control-flow history for the same hardware budget.
  - Second, by relaxing the ordering requirement among these unique PCs, Glider trains much faster than solutions that must learn the behavior of every distinct ordering in different predictor entries.

### Machine Learning in Computer Architecture

Extremely simple machine learning algorithms, such as the perceptron, have been used in dynamic branch prediction. For branch prediction, the use of perceptrons enabled the use of long branch histories, which inspired many future academic and commercial branch predictor implementations.

## Background

因为 Glider 是基于 Hawkeye 做的，所以需要了解 Hawkeye。

### The Hawkeye Cache

{% include figure.html path="assets/img/fig/Glider-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

The Hawkeye predictor is a binary classifier, whose goal is to predict whether data loaded by a memory access is likely to be cached or not by the optimal algorithm. Cache-friendly data is inserted in the cache with high priority, and cache-averse data is inserted with low priority.

Hawkeye uses the PC as a feature and maintains a table of counters to learn whether memory accesses by a given PC tend to be cache-friendly or cache-averse. While the Hawkeye Cache has been quite successful-it won the 2017 Cache Replacement Championship-its simple predictor achieves only 72.4% accuracy on a set of challenging benchmarks.

### Recurrent Neural Networks

LSTM is a widely used variant of RNNs that is designed to learn long, complex patterns within a sequence.
- Here, a complex pattern is one that can exhibit non-linear correlation among elements in a sequence. For example, noun-verb agreement in the English language can be complicated by prepositional phrases, so the phrase, "the idea of using many layers" is singular, even though the prepositional phrase ("of using many layers") is plural.

### Attention Mechanisms

LSTM has recently been coupled with attention mechanisms, which enable a sequence model to focus its attention on certain inputs.

这块不太明白：

{% include figure.html path="assets/img/fig/Glider-fig2.png" class="img-fluid rounded z-depth-1" zoomable=true %}

## Our Solution

We choose to identify loads by their PC instead of their memory address. 主要原因是 memory address 比 PC 多太多了。

### LSTM with Scaled Attention

{% include figure.html path="assets/img/fig/Glider-fig3.png" class="img-fluid rounded z-depth-1" zoomable=true %}

- Since PCs are **categorical in nature**, our model uses a **one-hot representation** for PCs, where the size of the PC vocabulary is the total number of PCs in the program.
- However, the one-hot representation is not ideal for neural networks because it treats each PC equally. So to create learnable representations for categorical features like the PC, we use an embedding layer before the LSTM layer.
- The LSTM layer learns caching behavior, and on top of the LSTM we add a scaled attention layer to learn correlations among PCs.

把 trace 切分成等长的 sequence，但为了每一段跑的时候都有 warmup，所以每次跑用实验前一半作 warmup，只采集后一半的结果做评估，并且每次的前一半是旧数据，后一半新数据。"Since the memory access trace is too long for LSTM-based models, we first preprocess the trace by slicing it into fixed-length sequences of length 2N. To avoid losing context for the beginning of each slice of the trace, we overlap consecutive sequences by half of the sequence length N."
- 比如 fix-size 为 2 (N = 2)，于是每次跑的序列长为 2N = 4，key id 序列大概是：1 2 3 4; 3 4 5 6; 5 6 7 8...

### Insights from Our LSTM Model

- Observation 1. Our model benefits from a long history of past PCs, in particular, the past 30 PCs. 本文发现 30 就差不多饱和了（见 Figure 14）。
- Observation 2. Our model can achieve good accuracy by **attending to just a few sources**.

{% include figure.html path="assets/img/fig/Glider-fig4.png" class="img-fluid rounded z-depth-1" zoomable=true %}

  - A moderate **increase in the scaling factor f forces sparsity in the attention weight vectors** but has minimal influence on accuracy.

{% include figure.html path="assets/img/fig/Glider-fig5.png" class="img-fluid rounded z-depth-1" zoomable=true %}

  - Without losing accuracy, the scaled attention weight distribution becomes biased towards a few source accesses, which shows that our model can make correct predictions based on just a few source memory accesses.

{% include figure.html path="assets/img/fig/Glider-fig6.png" class="img-fluid rounded z-depth-1" zoomable=true %}

- Observation 3. Prediction accuracy is largely **insensitive to the order of the sequence**.

{% include figure.html path="assets/img/fig/Glider-fig7.png" class="img-fluid rounded z-depth-1" zoomable=true %}

-> With an appropriate feature design, caching can be simplified from a sequence labeling problem to a binary classification problem.

### Integer SVM and a k-sparse Binary Feature

k-sparse binary 来代表忽略了顺序的 unique PC sequence

Fact 1. With binary features, the use of **gradient descent with learning rate γ = 1/n** for an integer n is **equivalent** to optimizing the following objective function with **learning rate 1**. 如果 initial weight vector 是 1/n learning rate 的 n 倍，learning rate 也变成 n 倍 (1/n * n = 1)，目标也变成 **n** - y * $$w^Tx$$，就等价了。好处是 ISVM 可以对 weight 做整数运算而不是浮点运算，进一步说，也就等价于了 perception 的复杂度。

### Hardware Design

对于每个 PC，都有一个 ISVM，这个 ISVM 有 16 个 weight。To find the 5 weights corresponding to the current contents of the PCHR, we create a 4-bit hash for each element in the PCHR (creating 5 indices in the range 0 to 15), and we retrieve the 5 weights at the corresponding indices.

{% include figure.html path="assets/img/fig/Glider-fig8.png" class="img-fluid rounded z-depth-1" zoomable=true %}

这里我比较好奇的是新增的 space overhead 和 latency overhead，本文说之后讨论。

magic number 开始了 :)

- Training:
  - Glider retrieves the weights corresponding to the current PC and the PCHR. The weights are incremented by 1 (not updated if their sum is above a certain threshold) if OPTgen determines that the line should have been cached; it is decremented otherwise. 
  - To find a good threshold, Glider’s predictor dynamically selects among a fixed set of thresholds (0, 30, 100, 300, and 3000). While this adaptive threshold provides some benefit for single-core workloads, the performance is largely insensitive to the choice of threshold for multi-core workloads.
- Prediction:
  - To make a prediction, the weights corresponding to the current PC and the PCHR are summed. If the summation is greater than or equal to a threshold (60 in our simulations), we predict that the line is cache-friendly and insert it with high priority (RRPV=07). If the summation is less than 0, we predict that the line is cache-averse and insert it with low priority (RRPV=7). For the remaining cases (sum between 0 and 60), we determine that the line is cache-friendly with a low confidence and insert it with medium priority (RRPV=2).

## EVALUATION

### Methodology

略

### Comparison of Offline Models and Online Models

{% include figure.html path="assets/img/fig/Glider-fig9.png" class="img-fluid rounded z-depth-1" zoomable=true %}

### Practicality of Glider vs. LSTM

{% include figure.html path="assets/img/fig/Glider-fig10.png" class="img-fluid rounded z-depth-1" zoomable=true %}

{% include figure.html path="assets/img/fig/Glider-fig11.png" class="img-fluid rounded z-depth-1" zoomable=true %}

- "Figure 15 shows that with offline training, our offline ISVM achieves good accuracy in one iteration, while the LSTM takes 10-15 iterations to converge. We also see that online models, such as, Perceptron and Hawkeye, converge fast but have the limited accuracy."
- "It’s clear that even with further compression techniques, deep learning models are still not ready for direct use in hardware predictors."

我不太明白多少 iteration 能代表多久……

### Learning High-Level Program Semantics

跳过了，不是很能看懂

## 评论

- 我还是不太理解 Glider 具体是如何 offline 转 online 的，似乎 offline 和 online 是绝对意义上的分离的，online 就是一个用了一些 magic number 的 ISVM（本文提到好多次其他工作需要 offline training 之后转 online 作为缺点）
- 前面有提到用了 Belady Algorithm，但实际上好像还是和 LRU 以及 100% accuracy 比较的
