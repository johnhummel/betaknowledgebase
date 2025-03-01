---
title: "Top N & Remain"
linkTitle: "Top N & Remain"
description: >
    Top N & Remain
---

```sql
CREATE TABLE top_with_rest
(
    `k` String,
    `number` UInt64
)
ENGINE = Memory;

INSERT INTO top_with_rest SELECT
    toString(intDiv(number, 10)),
    number
FROM numbers_mt(10000);
```

## Using UNION ALL

```sql
SELECT *
FROM
(
    SELECT
        k,
        sum(number) AS res
    FROM top_with_rest
    GROUP BY k
    ORDER BY res DESC
    LIMIT 10
    UNION ALL
    SELECT
        NULL,
        sum(number) AS res
    FROM top_with_rest
    WHERE k NOT IN (
        SELECT k
        FROM top_with_rest
        GROUP BY k
        ORDER BY sum(number) DESC
        LIMIT 10
    )
)
ORDER BY res ASC

┌─k───┬───res─┐
│ 990 │ 99045 │
│ 991 │ 99145 │
│ 992 │ 99245 │
│ 993 │ 99345 │
│ 994 │ 99445 │
│ 995 │ 99545 │
│ 996 │ 99645 │
│ 997 │ 99745 │
│ 998 │ 99845 │
│ 999 │ 99945 │
└─────┴───────┘
┌─k────┬──────res─┐
│ ᴺᵁᴸᴸ │ 49000050 │
└──────┴──────────┘
```

## Using arrays

```sql
WITH toUInt64(sumIf(sum, isNull(k)) - sumIf(sum, isNotNull(k))) AS total
SELECT
    (arrayJoin(arrayPushBack(groupArrayIf(10)((k, sum), isNotNull(k)), (NULL, total))) AS tpl).1 AS key,
    tpl.2 AS res
FROM
(
    SELECT
        toNullable(k) AS k,
        sum(number) AS sum
    FROM top_with_rest
    GROUP BY k
        WITH CUBE
    ORDER BY sum DESC
    LIMIT 11
)
ORDER BY res ASC

┌─key──┬──────res─┐
│ 990  │    99045 │
│ 991  │    99145 │
│ 992  │    99245 │
│ 993  │    99345 │
│ 994  │    99445 │
│ 995  │    99545 │
│ 996  │    99645 │
│ 997  │    99745 │
│ 998  │    99845 │
│ 999  │    99945 │
│ ᴺᵁᴸᴸ │ 49000050 │
└──────┴──────────┘
```

## Using window functions \(starting from 21.1\)

```sql
SET allow_experimental_window_functions = 1;

SELECT
    k AS key,
    If(isNotNull(key), sum, toUInt64(sum - wind)) AS res
FROM
(
    SELECT
        *,
        sumIf(sum, isNotNull(k)) OVER () AS wind
    FROM
    (
        SELECT
            toNullable(k) AS k,
            sum(number) AS sum
        FROM top_with_rest
        GROUP BY k
            WITH CUBE
        ORDER BY sum DESC
        LIMIT 11
    )
)
ORDER BY res ASC

┌─key──┬──────res─┐
│ 990  │    99045 │
│ 991  │    99145 │
│ 992  │    99245 │
│ 993  │    99345 │
│ 994  │    99445 │
│ 995  │    99545 │
│ 996  │    99645 │
│ 997  │    99745 │
│ 998  │    99845 │
│ 999  │    99945 │
│ ᴺᵁᴸᴸ │ 49000050 │
└──────┴──────────┘
```

