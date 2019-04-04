---
title: "3DF: Reactive Datalog for Datomic [talk]"
tags: 3DF Datalog Datomic
language: EN
author: niko
---

This is a conference talk Niko gave in cooperation with Frank McSherry, David, and Malte at ClojureD 2019.

<!--abstract-->
  
*This is a conference talk Niko gave in cooperation with Frank McSherry, David, and Malte at ClojureD 2019.*
  
## Abstract

3DF is a stream processing system which feeds off of Datomic’s transaction log and provides clients with the ability to register arbitrary Datalog queries, for which they will then continuously receive any changes to the result set. It does this efficiently by compiling Datalog queries to differential dataflows.

Using 3DF on top of Datomic provides a powerful, reactive interface to Datomic, making it an even more attractive choice for the real-time web. It also opens up Datomic to non-JVM runtimes and processes without a peer cache, without sacrificing performance. Finally, it hints at the possibility of significantly speeding up functional UI frameworks like D3 and React, because it allows these systems to skip their own change detection.

This talk will explore the „why?“, „how?“, and „what now?“ of working with a reactive database.

## Links

[Conference](https://clojured.de/archiv/schedule-2019/#nikolasGoebel)<br />
[Slides](https://github.com/comnik/talks/blob/raw/3df-reactive-datalog.pdf)

## Video

<iframe src="https://www.youtube.com/embed/CuSyVILzGDQ" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

<br />


