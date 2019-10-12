---
title: "About Time: An Introduction to Timely Dataflow [talk]"
tags: Timely Differential
language: EN
author:
  - malte
  - niko
---

This is a conference talk Malte and Niko gave at Data Council
Barcelona 2019.

<!--abstract-->

*This is a conference talk Malte and Niko gave at Data Council
Barcelona 2019.*
  
### Abstract

A central challenge of current stream processors is navigating the
trade-off between performance and consistency. Giving results too
early achieves low latency at the cost of missing late arrivals,
treading too carefully knocks us back into the world of batch
processing. The few stream processors that handle this well restrict
themselves to relatively simple computations (think MapReduce).

Timely Dataflow changes that. By rethinking how time should be
represented in a distributed system, it achieves lower latencies and
strong consistency, while allowing its users to express even complex
cyclic computations. Originally developed at Microsoft Research,
Timely has been refined over the last years by ETH Zurich's Systems
Group under the supervision of one of its creators, Frank McSherry.

In this talk, we give an introduction to the Timely stack, revealing
what makes it special, and how we use it in our professional
work. Using a real-world use case as a working example, we will guide
you through Timely's underlying dataflow model, its unique approach to
progress tracking, and give intuition for why it is able to outperform
even specialized systems in the wild. Along the way, we highlight some
of the more advanced aspects of the Timely ecosystem, and how your
organization can use it to supercharge its data-driven architectures.

### Links

[Conference](https://www.datacouncil.ai/talks/its-about-time-an-introduction-to-timely-dataflow?hsLang=en)<br />
[Slides](https://github.com/comnik/talks/raw/about-time-an-intro-to-timely-dataflow.pdf)

### Video

<iframe src="https://www.youtube.com/embed/ZN7nOwJTSZ0" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<br />

