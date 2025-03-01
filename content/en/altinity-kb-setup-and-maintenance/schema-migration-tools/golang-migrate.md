---
title: "golang-migrate"
linkTitle: "golang-migrate"
description: >
    golang-migrate
---

### `migrate`

`migrate` is a simple schema migration tool written in golang. No external dependencies are required \(like interpreter, jre\), only one platform-specific executable. [golang-migrate/migrate](https://github.com/golang-migrate/migrate)

`migrate` supports several databases, including ClickHouse \(support was introduced by [@kshvakov](https://github.com/kshvakov)\).

To store information about migrations state `migrate` creates one additional table in target database, by default that table is called `schema_migrations`.

#### Install:

[download](https://github.com/golang-migrate/migrate/releases) the `migrate` executable for your platform and put it to the folder listed in your %PATH.

```bash
---
title: "on Linux:"
linkTitle: "on Linux:"
description: >
    on Linux:
---
#wget https://github.com/golang-migrate/migrate/releases/download/v3.2.0/migrate.linux-amd64.tar.gz
wget https://github.com/golang-migrate/migrate/releases/download/v4.14.1/migrate.linux-amd64.tar.gz
tar -xzf migrate.linux-amd64.tar.gz 
mkdir -p ~/bin
mv migrate.linux-amd64 ~/bin/migrate
rm migrate.linux-amd64.tar.gz
```

#### Sample usage:

```bash
mkdir migrations
echo 'create table test(id UInt8) Engine = Memory;' > migrations/000001_my_database_init.up.sql
echo 'DROP TABLE test;' > migrations/000001_my_database_init.down.sql

---
title: "you can also auto-create file with new migrations with automatic numbering like that:"
linkTitle: "you can also auto-create file with new migrations with automatic numbering like that:"
description: >
    you can also auto-create file with new migrations with automatic numbering like that:
---
---
title: "> migrate create -dir migrations -seq -digits 6 -ext sql my_database_init"
linkTitle: "> migrate create -dir migrations -seq -digits 6 -ext sql my_database_init"
description: >
    > migrate create -dir migrations -seq -digits 6 -ext sql my_database_init
---
---
title: "edit migrations/000001_my_database_init.up.sql & migrations/000001_my_database_init.down.sql"
linkTitle: "edit migrations/000001_my_database_init.up.sql & migrations/000001_my_database_init.down.sql"
description: >
    edit migrations/000001_my_database_init.up.sql & migrations/000001_my_database_init.down.sql
---

➜ migrate -database 'clickhouse://localhost:9000' -path ./migrations up
1/u my_database_init (6.502974ms)

➜ migrate -database 'clickhouse://localhost:9000' -path ./migrations down
1/d my_database_init (2.164394ms)

---
title: "clears the database (use carefully - will not ask any confirmations)"
linkTitle: "clears the database (use carefully - will not ask any confirmations)"
description: >
    clears the database (use carefully - will not ask any confirmations)
---
➜ migrate -database 'clickhouse://localhost:9000' -path ./migrations drop
```

#### Connection string format

`clickhouse://host:port?username=user&password=qwerty&database=clicks`

| URL Query | Description |
| :--- | :--- |
| `x-migrations-table` | Name of the migrations table |
| `database` | The name of the database to connect to |
| `username` | The user to sign in as |
| `password` | The user's password |
| `host` | The host to connect to. |
| `port` | The port to bind to. |
| `secure` | to use a secure connection \(for self-signed also add `skip_verify=1`\) |

#### Replicated / Distributed / Cluster environments.

By default `migrate` create table `schema_migrations` with the following structure

```sql
CREATE TABLE schema_migrations (
  version UInt32,
  dirty UInt8,
  sequence UInt64
) ENGINE = TinyLog
```

That allows storing version of schema locally.

If you need to use `migrate` in some multi server environment \(replicated / cluster\) you should create `schema_migrations` manually with the same structure and with the appropriate Engine \(Replicated / Distributed\), otherwise, other servers will not know the version of the DB schema. As an alternative you can force the current version number on another server manually, like that:

```bash
migrate -database 'clickhouse://localhost:9000' -path ./migrations force 123456 # force version 123456
```

#### Known issues:

`could not load time location: unknown time zone Europe/Moscow in line 0:`  
  
It's happens due of missing tzdata package in migrate/migrate docker image of golang-migrate.  
There is 2 possible solutions: 

1. You can build your own golang-migrate image from official with tzdata package.
2. If you using it as part of your CI you can add installing tzdata package as one of step in ci before using golang-migrate.

Related GitHub issues:  
[https://github.com/golang-migrate/migrate/issues/494](https://github.com/golang-migrate/migrate/issues/494)  
[https://github.com/golang-migrate/migrate/issues/201](https://github.com/golang-migrate/migrate/issues/201)

Using database name in `x-migrations-table`  
  
1. Creates table with `database.table`  
2. When running migrations migrate actually uses database from query settings and encapsulate `database.table` as table name: ``other_database.`database.table```

