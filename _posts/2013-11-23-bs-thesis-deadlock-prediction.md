---
layout: post
title: "B.S. Thesis: deadlock prediction"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Please find my B.S. Thesis entitled [Dynamic Codeanalysis Based Deadlock Prediction][0] online.
It's hungarian, english abstract below.

Abstract
--------

The widespread of multicore processor architectures facilitates and speeds up the development of concurrent software architectures and algorithms. Being a rather demanding challenge the design of such systems, failure to design or implement a concurrent application often implies serious maintenance costs later. This thesis describes the common pitfalls such as race condition and deadlock and enumerates several solutions which counteract and prevent them. The solutions make use of the brand new and highly anticipated features of the C++11 standard, like the synchronization primitives supporting concurrency, the atomic memory objects and methods which exposes the interesting capabilities of transactional memory and the move semantics.

The second chapter introduces an analyzer tool based on dynamic code analysis which is able to predict the deadlocks of a system. This tool successfully provides diagnostics even in cases where the other standard solutions would require excess refactoring or would cause significant performance degradation.

[0]: http://www.scribd.com/doc/186541968/BSc-Thesis-Dynamic-Codeanalysis-Based-Deadlock-Prediction
