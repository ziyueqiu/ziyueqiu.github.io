---
layout:     post
title:      Glider (MICRO 19) 论文阅读
date:       2022-12-07
tags:
    - Cloud
categories: PaperReading
comments: true
---

[TMO: Transparent Memory Offloading in Datacenters](https://www.pdl.cmu.edu/ftp/NVM/tmo_asplos22.pdf)

第一篇 Deep Learning + Cache Replacement 走进现实

## Abstract

- a powerful LSTM learning model can in an offline setting provide better accuracy than current hardware predictors
- a simple online model that matches the offline model’s accuracy with orders of magnitude lower cost（Q:相对 cost 降低也不一定能放在硬件？）

Questions:

- 啥是 2nd Cache Replacement Championship
- 为什么多核的效果比单核明显？

## Introduction

deep learning 的好处是：The use of multiple layers is significant because it provides the ability to learn non-linear relationships at multiple levels of abstraction.

Challenge - "deep learning models are ill suited for use in hardware predictors"

- require enormous resources to train
  - offline training is much less effective for hardware predictors because (1) computer programs exhibit time-varying phase behavior, so prediction targets change as the program executes, and (2) the behavior of one program can differ wildly from that of another.
- too large to be implemented on a chip to make predictions
- slow, typically taking several milliseconds to produce a prediction, while hardware predictors typically need a prediction within nanoseconds.

Glider:

- design a powerful, unconstrained deep RNN model that is trained offline for individual programs
  - "sequence labeling problem": assign each element in a sequence of memory accesses a binary label that indicates whether the accessed data should be cached or not
  - -> attention-based LSTM as model
- interpret this offline model to reveal an important insightÐdescribed shortlyÐthat is useful for cache designers (???)
- use this insight to design a simple online model

Insights:

- a long history of past load instructions us beneficial
- the optimal decisions depend primarily on the presence of a few elements in the control-flow history, not the full ordered sequence (???)

Questions:

- "offline model should be able to reason about **time-ordered events**"???

## Related Work

### Cache Replacement

- Heuristic-Driven Solutions. They are customized for a limited class of known cache access patterns.
- Learning-Based Solutions. 
  - Hawkeye learns from the optimal solution for past accesses.
- Machine Learning-Based Solutions. Glider uses an unordered history of unique PCs.
  - First, because it does not include duplicate occurrences of the same PC, Glider uses an effectively longer control-flow history for the same hardware budget.
  - Second, by relaxing the ordering requirement among these unique PCs, Glider trains much faster than solutions that must learn the behavior of every distinct ordering in different predictor entries.

### Machine Learning in Computer Architecture

Extremely simple machine learning algorithms, such as the perceptron, have been used in dynamic branch prediction. For branch prediction, the use of perceptrons enabled the use of long branch histories, which inspired many future academic and commercial branch predictor implementations.

## Background

因为 Glider 是基于 Hawkeye 做的，所以需要了解 Hawkeye。

### The Hawkeye Cache

{% include figure.html path="assets/img/fig/Glider-fig1.png" class="img-fluid rounded z-depth-1" zoomable=true %}

The Hawkeye predictor is a binary classifier, whose goal is to predict whether data loaded by a memory access is likely to be cached or not by the optimal algorithm. Cache-friendly data is inserted in the cache with high priority, and cache-averse data is inserted with low priority.

### Recurrent Neural Networks

LSTM is a widely used variant of RNNs that is designed to learn long, complex patterns within a sequence.

### Attention Mechanisms

LSTM has recently been coupled with attention mechanisms, which enable a sequence model to focus its attention on certain inputs.

## Our Solution

We choose to identify loads by their PC instead of their memory address.（原因略）

