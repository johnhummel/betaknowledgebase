---
title: "Parameterized views"
linkTitle: "Parameterized views"
description: >
    Parameterized views
---

Custom settings allows to emulate parameterized views.

You need to enable custom settings and define any prefixes for settings.

```markup
$ cat /etc/clickhouse-server/config.d/custom_settigs_prefix.xml
<?xml version="1.0" ?>
<yandex>
    <custom_settings_prefixes>my,my2</custom_settings_prefixes>
</yandex>

$ service clickhouse-server restart
```

Now you can set settings as any other settings, and query them using **getSetting\(\)** function.

```sql
SET my2_category='hot deals';

SELECT getSetting('my2_category');
┌─getSetting('my2_category')─┐
│ hot deals                  │
└────────────────────────────┘

-- you can query ClickHouse settings as well
SELECT getSetting('max_threads')
┌─getSetting('max_threads')─┐
│                         8 │
└───────────────────────────┘
```

Now we can create a view

```sql
CREATE VIEW my_new_view AS
SELECT *
FROM deals
WHERE category_id IN
(
    SELECT category_id
    FROM deal_categories
    WHERE category = getSetting('my2_category')
);
```

And query it

```sql
SELECT *
FROM my_new_view
SETTINGS my2_category = 'hot deals';
```

© 2021 Altinity Inc. All rights reserved.

