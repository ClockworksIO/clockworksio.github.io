---
title: Two More Kafka Superpowers
tags: Kafka kplex
language: EN
author:
  - niko
  - malte
  - david
---

@TODO Using kplex to de- and re-normalize Kafka topics on-the-fly.

<!--abstract-->

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
