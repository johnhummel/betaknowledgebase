---
title: "System tables eat my disk"
linkTitle: "System tables eat my disk"
description: >
    System tables eat my disk
---

> **Note 1:** System database stores virtual tables \(**parts**, **tables,** **columns, etc.**\) and \***\_log** tables.
>
> Virtual tables do not persist on disk. They reflect ClickHouse memory \(c++ structures\). They cannot be changed or removed.
>
> Log tables are named with postfix \***\_log** and have the MergeTree engine.
>
> You can drop / rename / truncate \***\_log** tables at any time. ClickHouse will recreate them in about 7 seconds \(flush period\).

> **Note 2:** Log tables with numeric postfixes \(\_1 / 2 / 3 ...\) `query_log_1 query_thread_log_3` are results of Clickhouse upgrades. When a new version of Clickhouse starts and discovers that a system log table's schema is incompatible with a new schema, then Clickhouse renames the old query\_log table to the name with the prefix and creates a table with the new schema. You can drop such tables if you don't need such historic data.

## You can disable all / any of them: <a id="Systemtableseatmydisk-Youcandisableall/anyofthem:"></a>

Do not create log tables at all \(a restart is needed for these changes to take effect\).

```markup
$ cat /etc/clickhouse-server/config.d/z_log_disable.xml
<?xml version="1.0"?>
<yandex>
    <asynchronous_metric_log remove="1"/>
    <metric_log remove="1"/>
    <part_log remove="1" />
    <query_log remove="1" />
    <query_thread_log remove="1" />
    <text_log remove="1" />
    <trace_log remove="1"/>
</yandex>
```

We do not recommend removing `query_log` and `query_thread_log` as queries' logging can be easily turned off without a restart through user profiles:

```markup
$ cat /etc/clickhouse-server/users.d/z_log_queries.xml
<yandex>
    <profiles>
        <default>
            <log_queries>0</log_queries>
            <log_query_threads>0</log_query_threads>
        </default>
    </profiles>
</yandex>
```

Hint: `z_log_disable.xml` is named with **z\_** in the beginning, it means this config will be applied the last and will override all other config files with these sections \(config are applied in alphabetical order\).

## You can configure TTL: <a id="Systemtableseatmydisk-YoucanconfigureTTL:"></a>

Example for `query_log`. It drops partitions with data older than 14 days:

```markup
$ cat /etc/clickhouse-server/config.d/query_log_ttl.xml
<?xml version="1.0"?>
<yandex>
    <query_log>
        <database>system</database>
        <table>query_log</table>
        <engine>ENGINE = MergeTree PARTITION BY (event_date) 
                ORDER BY (event_time) 
                TTL event_date + INTERVAL 14 DAY DELETE
                SETTINGS ttl_only_drop_parts=1
        </engine>
        <flush_interval_milliseconds>7500</flush_interval_milliseconds>
    </query_log>
</yandex>
```

After that you need to restart ClickHouse and drop or rename the existing system.query\_log table, then CH creates a new table with these settings.

```sql
RENAME TABLE system.query_log TO system.query_log_1;
```

Important part here is a daily partitioning `PARTITION BY (event_date)` and `ttl_only_drop_parts=1`. In this case ClickHouse drops whole partitions. Dropping of partitions is very easy operation for CPU / Disk I/O.

Usual TTL \(without `ttl_only_drop_parts=1`\) is heavy CPU / Disk I/O consuming operation which re-writes data parts without expired rows.

You can add TTL without ClickHouse restart \(and table dropping or renaming\):

```sql
ALTER TABLE system.query_log MODIFY TTL event_date + INTERVAL 14 DAY;

ALTER TABLE system.query_log MODIFY SETTING ttl_only_drop_parts = 1;
```

But in this case ClickHouse will drop only whole monthly partitions \(will store data older than 14 days\).

## One more way to configure TTL for system tables <a id="Systemtableseatmydisk-OnemorewaytoconfigureTTLforsystemtables"></a>

This way just adds TTL to a table and leaves monthly \(default\) partitioning \(will store data older than 14 days\).

```markup
$ cat /etc/clickhouse-server/config.d/query_log_ttl.xml
<?xml version="1.0"?>
<yandex>
    <query_log>
        <database>system</database>
        <table>query_log</table>
        <ttl>event_date + INTERVAL 30 DAY DELETE</ttl>
    </query_log>
</yandex>
```

After that you need to restart ClickHouse and drop or rename the existing system.query\_log table, then CH creates a new table with this TTL setting.

## You can disable logging on a session level or in user’s profile \(for all or specific users\): <a id="Systemtableseatmydisk-Youcandisableloggingonasessionlevelorinuser&#x2019;sprofile(forallorspecificusers):"></a>

But only for logs generated on session level \(`query_log` / `query_thread_log`\)

In this case a restart is not needed.

Let’s disable query logging for all users \(profile = default, all other profiles inherit it\).

```markup
cat /etc/clickhouse-server/users.d/log_queries.xml
<yandex>
    <profiles>
        <default>
            <log_queries>0</log_queries>
            <log_query_threads>0</log_query_threads>
        </default>
    </profiles>
</yandex>
```

© 2021 Altinity Inc. All rights reserved.

