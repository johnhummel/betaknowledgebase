---
title: "Kafka parallel consuming"
linkTitle: "Kafka parallel consuming"
description: >
    Kafka parallel consuming
---

For very large topics when you need more parallelism \(especially on the insert side\) you may use several tables with the same pipeline \(pre 20.9\) or enable `kafka_thread_per_consumer` \(after 20.9\).

```text
kafka_num_consumers = N,
kafka_thread_per_consumer=1
```

Notes:

* the inserts will happen in parallel \(without that setting inserts happen linearly\)
* enough partitions are needed.

Before increasing `kafka_num_consumers` with keeping `kafka_thread_per_consumer=0` may improve consumption & parsing speed, but flushing & committing still happens by a single thread there \(so inserts are linear\).

© 2021 Altinity Inc. All rights reserved.

