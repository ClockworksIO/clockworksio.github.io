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
record. Immutability, we might say, cuts both ways. 

In the following we focus on partitioning — arguably the highest
impact decision in any Kafka setup — from three different
perspectives: the logical, the physical, and the modeling point of
view. We also introduce `kplex`, a tool we built for whenever we need
to bend an entrenched Kafka setup in ways it does not want to go.

## Partitioning - The Logical Perspective

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

**Fully Sequential** We need to persist a database transaction log in
Kafka, i.e. a sequence of transaction data annotated with logical
transaction times `t0`, `t1`, `t2`, and so forth. For such a topic,
any downstream consumer will have to see *all* records up to some `t*`
and — crucially — see them *in transaction time order*. We therefore
have no choice but to use a single partition for this topic.

**Embarrasingly Parallel** We need to compress a stream of image data
and upload them to a blob store. Here, the processing time for each
record is comparatively high, but any individual record is fully
self-contained and can thus be processed by a stateless consumer. In
this scenario, we are free to choose the number of topics entirely
based on the throughput that we want this system to achieve.

Many Kafka use cases fall somewhere closer to the center of this
spectrum, exhibiting more granular "consistency domains" within which
records must be presented in order. Examples for such domains are the
subset of records affecting an individual user or those originating
from a specific geographic region.

**Sharded** [TODO]

## Partitioning - The Physical Perspective

Once we know how much concurrency we can hope for without giving up
consistency

Uniformly partitioned records prevent hotspots
and thus ensure smooth operations. On the other hand, partitioning by
domain attributes is crucial for consistent, stateful processing.

```
# pro/contra uniform partitioning
+ no hotspots
- no consistency
```

[source: https://blog.newrelic.com/engineering/effective-strategies-kafka-topic-partitioning/]

## Partitioning - The Modeling Perspective

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

## Partitioning Decoupled

Summarizing the above (and glossing over Kafka's idiosyncracies, most
of which could reasonably be fixed in the future), we have seen now
how throughput and skew considerations alone would lead us to using
many physical partitions and distribute records randomly amongst
them. It is only our need for ordering guarantees which forces us to
make compromises. This raises an uncomfortable issue.

Kafka decouples data producers from data consumers, by acting as the
unified "plumbing" platform between them. This is highly attractive to
organizations, because it allows them to support many different teams
on a shared, coherent view of their business. *Consistency, however,
is a joint property of producer, storage, and consumer.* Therefore, no
matter how carefully we make these decisions, they will always
interfere with some valid use cases later on. 

We realized that in order to satisfy the trimuvirate of throughput,
skew, and consistency, we would have to *decouple physical from
logical partitioning* as much as possible, while <u>preserving
ordering guarantees</u> in the process. That is why we built
[kplex](https://www.clockworks.io/kplex/).
