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
Kafka setup — from three different perspectives, and introduce kplex,
a tool we built for whenever we need to bend an entrenched Kafka setup
in ways it does not want to go.

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
[kplex](https://www.clockworks.io/kplex/), a tool we built to
repartition Kafka topics consistently and on-the-fly, allowing you to
unlock consumer concurrency and perform correct stateful processing
across partitions.

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

## Unlocking Consumer Concurrency

Let's look at . We compress a stream of
image data and upload them to a blob store. More generally we need to
perform a (relatively) expensive computation for each input record,
but records can be considered in isolation and in an arbitrary
order. In this scenario we *should* be able to employ plenty of
threads

@TODO (example: transaction log, computing commutative aggregate)

## Reading Consistently From Multiple Partitions

@TODO (example: transaction log uniformly distributed, computing state
of the database)

@TODO "how can this be a scalability benefit?" -> not yet, that's
where the rest of incremental view maintenance comes in

[^partition-limit]: Previously in the hundreds, nowadays [in the thousands](https://www.confluent.io/blog/apache-kafka-supports-200k-partitions-per-cluster).
[^partition-performance]: [Jun Rao, "How to choose the number of topics/partitions in a Kafka cluster?"](https://www.confluent.io/blog/how-choose-number-topics-partitions-kafka-cluster)
[^newrelic]: [Amy Boyle, "Effective Strategies for Kafka Topic Partitioning"](https://blog.newrelic.com/engineering/effective-strategies-kafka-topic-partitioning/)
