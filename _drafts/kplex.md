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

Like any technology considered core infrastructure, Kafka forces its
users to make certain trade-offs. What sides we take in these trades
becomes increasingly harder to change as data accumulates. This is
exacerbated by Kafka's distinguished role as an immutable historic
record. Immutability — we might say — cuts both ways. Today we discuss
three high-impact architectural decisions we had to make in our Kafka
setups, and introduce `kplex`, a tool we built for whenever we needed
to change our minds after the fact.

## How many partitions per topic?

Partitions are both the unit of concurrency and of consistency in
Kafka. The more partitions we have, the more consumer instances we can
bring to bear in parallel, increasing throughput. On the other hand,
records that need to stay in a fixed order must go to the same
partition.

Kafka would like us to keep the number of partitions within reasonable
limits (previously in the hundreds, nowadays [in the
thousands](https://www.confluent.io/blog/apache-kafka-supports-200k-partitions-per-cluster))
for [various performance-related
reasons]((https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster)). However
it is the ordering guarantees provided by partitions that impose much
more stringent constraints, as the following two extremes will
highlight.

**Scenario A** We need to persist a database transaction log in Kafka,
i.e. a sequence of transaction data annotated with logical transaction
times `t0`, `t1`, `t2`, and so forth. For such a topic, any downstream
consumer will have to see *all* records up to some `t*` and —
crucially — see them *in transaction time order*. We therefore have no
choice but to use a single partition for this topic.

**Scenario B** We need to relay a stream of alerts to a downstream
system, that notifies alerted users via mail. Any individual alert is
fully self-contained and can thus be processed by a stateless
consumer. In this scenario, we are free to choose the number of topics
entirely based on the throughput that we want this system to achieve.

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

## kplex

No matter how carefully we make these decisions, they will always
interfere with some use cases later on. Can we build tools to simulate
all the other choices after the fact? We realized that what was
required to solve all of these challenges, was to *decouple physical
from logical partitioning, while <u>preserving ordering</u>
guarantees*.

That is why we built [kplex](https://www.clockworks.io/kplex/).
