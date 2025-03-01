---
title: "Cluster Configuration FAQ"
linkTitle: "Cluster Configuration FAQ"
description: >
    Cluster Configuration FAQ
---

#### ClickHouse does not start, some other unexpected behavior happening.

Check clickhouse logs, they are your friends:  
  
tail -n 1000 /var/log/clickhouse-server/clickhouse-server.err.log \| less  
tail -n 10000 /var/log/clickhouse-server/clickhouse-server.log \| less

### How Do I Restrict Memory Usage?

See [our knowledge base article](https://kb.altinity.com/altinity-kb-setup-and-maintenance/altinity-kb-memory-configuration-settings)  and [official documentation](https://clickhouse.tech/docs/en/operations/settings/query-complexity/#settings_max_memory_usage) for more information.

### ClickHouse died during big query execution

Misconfigured clickhouse can try to allocate more RAM than is available on the system. 

In that case an OS component called oomkiller can kill the clickhouse process. 

That event leaves traces inside system logs \(can be checked by running dmesg command\). 

### How Do I make huge ‘Group By’ queries use less RAM?

Enable on disk GROUP BY \(it is slower, so is disabled by default\)

Set [max\_bytes\_before\_external\_group\_by](https://clickhouse.tech/docs/en/operations/settings/query-complexity/#settings-max_bytes_before_external_group_by) to a value about 70-80% of your max\_memory\_usage value.

### Data returned in chunks by clickhouse-client 

See https://kb.altinity.com/altinity-kb-interfaces/altinity-kb-clickhouse-client

### I Can’t Connect From Other Hosts.  What do I do?

Check the &lt;listen&gt; settings in config.xml. Verify that the connection can connect on both IPV4 and IPV6.

