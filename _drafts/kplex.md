---
title: kplex - A Kafka Superpower
tags: Timely Kafka
language: EN
author:
  - niko
  - malte
  - david
---

@TODO

<!--abstract-->

Like any technology considered core infrastructure, Kafka makes a
number of trade-offs in order to offer capabilities that other systems
can't. [TODO...] Immutability, we might say, cuts both ways. In this
work we discuss three high-impact architectural decisions we had to
make in our Kafka setups, and introduce `kplex`, a tool we built for
when we [TODO...]

## How many partitions per topic?

Kafka encourages you to choose a reasonable number of partitions for
your topics. But computers are fast and more partitions can unlock
more concurrency.

```
# pro/contra lots of partitions
+ concurrency
+ distribution
- consistency
- performance
```

[source: https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster]

## How many types per topic?

Stateful processing is easier when everything is in a single topic,
especially when order matters. But normalization allows for scale,
simpler schemas, and granular retention policies.

```
# pro/contra denormalization
+ consistency
+ simple consumer logic (only 1 poll)
- granularity of policies
- granularity of consumption
- schema complexity
```

[source: https://martin.kleppmann.com/2018/01/18/event-types-in-kafka-topic.html]

## How to partition the data?

Uniformly partitioned records prevent hotspots and thus ensure smooth
operations. On the other hand, partitioning by domain attributes is
crucial for consistent, stateful processing.

```
# pro/contra uniform partitioning
+ no hotspots
- no consistency
```

[source: https://blog.newrelic.com/engineering/effective-strategies-kafka-topic-partitioning/]

## kplex

No matter how carefully we make these decisions, they will always
interfere with some use cases later on. Can we build tools to simulate
all the other choices after the fact? We realized that what was
required to solve all of these challenges, was to *decouple physical
from logical partitioning, while <u>preserving ordering</u>
guarantees*.

That is why we built [kplex](https://www.clockworks.io/kplex/).
