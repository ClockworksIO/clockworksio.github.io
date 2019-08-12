---
title: Two Kafka Superpowers
tags: Kafka kplex
language: EN
author:
  - niko
  - malte
  - david
---

We discuss partitioning — arguably the highest impact decision in any
Kafka setup — and introduce kplex, a tool for repartitioning Kafka
topics consistently and on-the-fly, allowing you to unlock concurrency
and perform correct stateful processing across partitions.

<!--abstract-->

Like any technology considered core infrastructure, Kafka forces its
users to make certain trade-offs. What sides we take in these trades
becomes increasingly harder to change as data accumulates. This is
exacerbated by Kafka's distinguished role as an immutable historic
record. Immutability, we might say, cuts both ways. 

In the following we look at partitioning — arguably the highest impact
decision in any Kafka setup — from two different perspectives: the
physical (concerned with scalability) and the logical (concerned with
correctness). We notice that an optimal, correctness-preserving
partitioning strategy *depends on what consumers will do* with the
data, thus impeding later use cases. We introduce
[kplex](https://www.clockworks.io/kplex/), a tool for repartitioning
Kafka topics consistently and on-the-fly, allowing you to unlock
consumer concurrency and perform correct stateful processing across
partitions.

## Partitioning - The Physical Perspective

Clusters are happy as long as each machine is only asked to bear load
in proportion to its relative capabilities. I.e., for a cluster made
up of identical machines, we want data to be distributed
uniformly. Machines that are asked to do more than their fair share,
tend to act up, or worse, get in the way of their peers.

In an ideal setting, taking into account only the physical
perspective, we would have the freedom to distribute data exclusively
according to the relative capabilities of each machine in our
cluster. Doing so prevents hotspots and ensures smooth scaling as we
add machines.

Unfortunately this requirement is often at odds with the constraints
imposed by the (arguably much more important!) need to produce correct
outputs.

## Partitioning - The Logical Perspective

Partitions are both the unit of concurrency and of consistency in
Kafka. The more partitions we have, the more consumer instances we can
bring to bear in parallel, increasing throughput. On the other hand,
records that need to be consumed in aggregate, in a specific order, or
both must go to the same partition.

Kafka would like us to keep the number of partitions within reasonable
limits[^partition-limit] for various performance-related
reasons[^partition-performance]. However it is the ordering guarantees
provided by partitions that impose much more stringent constraints, as
the following two extremes will highlight.

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

In an ideal setting, from a logical point of view, we therefore want
the freedom to assign each available consistency domain to its own
physical partition for maximum throughput — while still making sure
that a consumer sees all records for any specific key, and in the
exact order they were produced in.

## Partitioning Decoupled

Summarizing the above (and glossing over some of the idiosyncracies in
Kafka's current implementation, most of which could reasonably be
fixed in the future), we have seen now how throughput and skew
considerations alone would lead us to using many physical partitions
and distribute records randomly between them. It is only our need for
ordering guarantees which forces us to make compromises. This raises
an uncomfortable issue.

Kafka decouples data producers from data consumers, by acting as the
unified "plumbing" platform between them. This is highly attractive to
organizations, because it allows them to support many different teams
on a shared, coherent view of their business. *Consistency, however,
is a joint property of producer, storage, and consumer.* Therefore, no
matter how carefully we make these decisions, they will always
interfere with some valid use cases later on.

> [...] the partitioning strategy for your producers depends on what
> your consumers will do with the data.
>
> - Amy Boyle, "Effective Strategies for Kafka Topic Partitioning"[^newrelic]

In order to satisfy the trimuvirate of throughput, skew, and
consistency, we will have to *decouple physical from logical
partitioning* as much as possible, while <u>preserving ordering
guarantees</u> in the process. We built
[kplex](https://www.clockworks.io/kplex/) to do precisely that.

Specifically, we encounter two types of mismatch: (1) strong physical
guarantees supporting weak logical requirements — here the physical
distribution *limits the concurrency* that could otherwise be applied
to the computation, and (2) weak physical guarantees thwarting strong
logical requirements — here the physical distribution *makes it
impossible* to do consistent, stateful processing.

`kplex` solves both of these.

## Repartitioning a topic on-the-fly

Let's start with a simple example. `ksql-datagen`[^datagen] is a handy
tool for generating synthetic Kafka topics. We use this to populate
three partitions of a `pageviews` topic, each containing records not
unlike the following sample:

``` json
{"viewtime":1565606197468,"userid":"User_6","pageid":"Page_62"}
{"viewtime":1565606197566,"userid":"User_9","pageid":"Page_60"}
{"viewtime":1565606197821,"userid":"User_4","pageid":"Page_98"}
{"viewtime":1565606198058,"userid":"User_7","pageid":"Page_14"}
{"viewtime":1565606198622,"userid":"User_1","pageid":"Page_63"}
```

Pretend that we initially choose `pageid` as the partitioning key,
thus making sure that all pageview records for any specific page end
up on the same partition and remain in the order they were produced in
(which corresponds to the `viewtime` order). 

While this suited our initial use cases perfectly, we might want to
write a consumer that looks at the history of pageviews *of each
individual user*. This is problematic, because pageview records for
any individual user are strewn across all of the physical
partitions. We therefore want to consume all partitions in parallel,
reshuffling records by `userid` as we go, all the while preserving
`viewtime` order. This is captured by the following `kplex` job:

``` json
{
  "workers": 3,
  "kafka": { "broker": "localhost:9092" },
  "topics": {
    "pageviews": {
      "max_delay_ms": 30000,
      "polling_interval": {"secs": 1, "nanos": 0}
    }
  },
  "derive": {
    "pageviews_by_user": {
      "from": "pageviews",
      "key": {"Pointer": "/userid"},
      "timestamp": {"Pointer": "/viewtime"},
      "order": "TimeOrder",
      "output": {
        "VirtualPartitions": { "count": 9 }
      }
    }
  }
}
```

Let's try it out, before going through it piece-by-piece.



## Unlocking Consumer Concurrency

## Reading Consistently From Multiple Partitions

@TODO (example: transaction log uniformly distributed, computing state
of the database)

@TODO "how can this be a scalability benefit?" -> not yet, that's
where the rest of incremental view maintenance comes in

[^partition-limit]: Previously in the hundreds, nowadays [in the thousands](https://www.confluent.io/blog/apache-kafka-supports-200k-partitions-per-cluster).
[^partition-performance]: [Jun Rao, "How to choose the number of topics/partitions in a Kafka cluster?"](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster)
[^newrelic]: [Amy Boyle, "Effective Strategies for Kafka Topic Partitioning"](https://blog.newrelic.com/engineering/effective-strategies-kafka-topic-partitioning/)
[^datagen]: [ksql-datagen](https://docs.confluent.io/current/ksql/docs/tutorials/generate-custom-test-data.html)
