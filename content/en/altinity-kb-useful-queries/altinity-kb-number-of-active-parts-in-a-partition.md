---
title: "Number of active parts in a partition"
linkTitle: "Number of active parts in a partition"
description: >
    Number of active parts in a partition
---

Q: Why do I have several active parts in a partition? Why Clickhouse does not merge them immediately?

A: CH does not merge parts by time.  
Merge scheduler selects parts by own algorithm based on the current node workload / number of parts / size of parts.

CH merge scheduler balances between a big number of parts and a wasting resources on merges.  
Merges are CPU/DISK IO expensive. If CH will merge every new part then all resources will be spend on merges and will no resources remain on queries \(selects \).

© 2021 Altinity Inc. All rights reserved.

