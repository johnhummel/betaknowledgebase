---
title: "Server config files"
linkTitle: "Server config files"
description: >
    Server config files
---

* [Сonfig management \(recommended structure\)](altinity-kb-server-config-files.md#Serverconfigfiles-Сonfigmanagement%28recommendedstructure%29)
* [Settings & restart](altinity-kb-server-config-files.md#Serverconfigfiles-Settings&restart)
* [Dictionaries](altinity-kb-server-config-files.md#Serverconfigfiles-Dictionaries)
* [incl attribute & metrica.xml](altinity-kb-server-config-files.md#Serverconfigfiles-inclattribute&metrica.xml)
* [Multiple Clickhouse instances at one host](altinity-kb-server-config-files.md#Serverconfigfiles-MultipleClickhouseinstancesatonehost)
* [preprocessed\_configs](altinity-kb-server-config-files.md#Serverconfigfiles-preprocessed_configs)

## Сonfig management \(recommended structure\) <a id="Serverconfigfiles-&#x421;onfigmanagement(recommendedstructure)"></a>

Clickhouse server config consists of two parts server settings \(config.xml\) and users settings \(users.xml\).

By default they are stored in the folder **/etc/clickhouse-server/** in two files config.xml & users.xml.

We suggest never change vendor config files and place your changes into separate .xml files in sub-folders. This way is easier to maintain and ease Clickhouse upgrades.

**/etc/clickhouse-server/users.d** – sub-folder for user settings.

**/etc/clickhouse-server/config.d** – sub-folder for server settings.

**/etc/clickhouse-server/conf.d** – sub-folder for any \(both\) settings.

File names of your xml files can be arbitrary but they applied in alphabetical order.

Examples:

```markup
$ cat /etc/clickhouse-server/config.d/macros.xml
<?xml version="1.0" ?>
<yandex>
  <macros>
    <cluster>test</cluster>
    <replica>host22</replica>
    <shard>0</shard>
    <server_id>41295</server_id>
    <server_name>host22.server.com</server_name>
  </macros>
</yandex>

cat /etc/clickhouse-server/config.d/zoo.xml
<?xml version="1.0" ?>
<yandex>
  <zookeeper>
    <node>
      <host>localhost</host>
      <port>2181</port>
    </node>
  </zookeeper>
  <distributed_ddl>
    <path>/clickhouse/test/task_queue/ddl</path>
  </distributed_ddl>
</yandex>

cat /etc/clickhouse-server/users.d/enable_access_management_for_user_default.xml
<?xml version="1.0" ?>
<yandex>
  <users>
    <default>
      <access_management>1</access_management>
    </default>
  </users>
</yandex>
```

BTW, you can define any macro in your configuration and use them in Zookeeper paths

```text
 ReplicatedMergeTree('/clickhouse/{cluster}/tables/my_table','{replica}')
```

or in your code using function getMacro:

```sql
CREATE OR REPLACE VIEW srv_server_info 
SELECT (SELECT getMacro('shard')) AS shard_num,
       (SELECT getMacro('server_name')) AS server_name,
       (SELECT getMacro('server_id')) AS server_key
```

Settings can be appended to an XML tree \(default behaviour\) or replaced or removed.

Example how to delete **tcp\_port** & **http\_port** defined on higher level in the main config.xml \(it disables open tcp & http ports if you configured secure ssl\):

```markup
cat /etc/clickhouse-server/config.d/disable_open_network.xml
<?xml version="1.0"?>
<yandex>
  <http_port remove="1"/>
  <tcp_port remove="1"/>
</yandex>
```

Example how to replace **remote\_servers** section defined on higher level in the main config.xml \(it allows to remove default test clusters.

```markup
<?xml version="1.0" ?>
<yandex>
  <remote_servers replace="1">
    <mycluster>
      ....
    </mycluster>
  </remote_servers>
</yandex>
```

## Settings & restart <a id="Serverconfigfiles-Settings&amp;restart"></a>

All users settings don’t need server restart but applied on connect. User need to reconnect to Clickhouse server.

Most of server settings applied only on a server start, except sections:

1. &lt;remote\_servers&gt; \(cluster config\)
2. &lt;dictionaries&gt; \(ext.dictionaries\)
3. &lt;max\_table\_size\_to\_drop&gt; & &lt;max\_partition\_size\_to\_drop&gt;

## Dictionaries <a id="Serverconfigfiles-Dictionaries"></a>

We suggest to store each dictionary description in a separate \(own\) file in a **/etc/clickhouse-server/dict** sub-folder.

```markup
$ cat /etc/clickhouse-server/dict/country.xml
<?xml version="1.0"?>
<dictionaries>
  <dictionary>
    <name>country</name>
    <source>
      <http>
      ...
  </dictionary>
</dictionaries>
```

and add to the configuration

```markup
$ cat /etc/clickhouse-server/config.d/dictionaries.xml
<?xml version="1.0"?>
<yandex>
  <dictionaries_config>dict/*.xml</dictionaries_config>
  <dictionaries_lazy_load>true</dictionaries_lazy_load>
</yandex>
```

**dict/\*.xml** – relative path, servers seeks files in the folder **/etc/clickhouse-server/dict**. More info in [Multiple Clickhouse instances](altinity-kb-server-config-files.md#Multiple-Clickhouse-instances).

## incl attribute & metrica.xml <a id="Serverconfigfiles-inclattribute&amp;metrica.xml"></a>

**incl** attribute allows to include some XML section from a special **include** file multiple times.

By default **include** file is **/etc/metrika.xml**. You can use many include files for each XML section.

For example to avoid repetition of user/password for each dictionary you can create an XML file:

```markup
$ cat /etc/clickhouse-server/dict_sources.xml 
<?xml version="1.0"?>
<yandex>
  <mysql_config>
      <port>3306</port>
      <user>user</user>
      <password>123</password>
      <replica>
        <host>mysql_host</host>
        <priority>1</priority>
      </replica>
      <db>my_database</db>
  </mysql_config>
</yandex>
```

Include this file:

```markup
$ cat /etc/clickhouse-server/config.d/dictionaries.xml
<?xml version="1.0"?>
<yandex>
  ...
  <include_from>/etc/clickhouse-server/dict_sources.xml</include_from>
</yandex>
```

And use in dictionary descriptions \(**incl="mysql\_config"**\):

```markup
$ cat /etc/clickhouse-server/dict/country.xml
<?xml version="1.0"?>
<dictionaries>
  <dictionary>
    <name>country</name>
    <source>
        <mysql incl="mysql_config">
            <table>my_table</table>
            <invalidate_query>select max(id) from my_table</invalidate_query>
        </mysql>
    </source>        
      ...
  </dictionary>
</dictionaries>
```

## Multiple Clickhouse instances at one host <a id="Serverconfigfiles-MultipleClickhouseinstancesatonehost"></a>

By default Clickhouse server configs are in **/etc/clickhouse-server/** because clickhouse-server runs with a parameter **--config-file /etc/clickhouse-server/config.xml**

**config-file** is defined in startup scripts:

* **/etc/init.d/clickhouse-server** – init-V
* **/etc/systemd/system/clickhouse-server.service** – systemd

Clickhouse uses the path from **config-file** parameter as base folder and seeks for other configs by relative path. All sub-folders **users.d / config.d** are relative.

You can start multiple **clickhouse-server** each with own **--config-file.**

For example:

```text
/usr/bin/clickhouse-server --config-file /etc/clickhouse-server-node1/config.xml
  /etc/clickhouse-server-node1/  config.xml ... users.xml
  /etc/clickhouse-server-node1/config.d/disable_open_network.xml
  /etc/clickhouse-server-node1/users.d/....

/usr/bin/clickhouse-server --config-file /etc/clickhouse-server-node2/config.xml
  /etc/clickhouse-server-node2/   config.xml ... users.xml
  /etc/clickhouse-server-node2/config.d/disable_open_network.xml
  /etc/clickhouse-server-node2/users.d/....
```

If you need to run multiple servers for CI purposes you can combine all settings in a single fat XML file and start ClickHouse without config folders/sub-folders.

```text
/usr/bin/clickhouse-server --config-file /tmp/ch1.xml
/usr/bin/clickhouse-server --config-file /tmp/ch2.xml
/usr/bin/clickhouse-server --config-file /tmp/ch3.xml
```

Each ClickHouse instance must work with own **data-folder** and **tmp-folder**.

By default ClickHouse uses **/var/lib/clickhouse/**. It can be overridden in path settings

```markup
<path>/data/clickhouse-ch1/</path>

<tmp_path>/data/clickhouse-ch1/tmp/</tmp_path>

<user_files_path>/data/clickhouse-ch1/user_files/</user_files_path>
  <local_directory>
    <path>/data/clickhouse-ch1/access/</path>
  </local_directory>

<format_schema_path>/data/clickhouse-ch1/format_schemas/</format_schema_path>
```

## preprocessed\_configs <a id="Serverconfigfiles-preprocessed_configs"></a>

Clickhouse server watches config files and folders. When you change, add or remove XML files Clickhouse immediately assembles XML files into a combined file. These combined files are stored in **/var/lib/clickhouse/preprocessed\_configs/** folders.

You can verify that your changes are valid by checking **/var/lib/clickhouse/preprocessed\_configs/config.xml**, **/var/lib/clickhouse/preprocessed\_configs/users.xml**.

If something wrong with with your settings e.g. unclosed XML element or typo you can see alerts about this mistakes in **/var/log/clickhouse-server/clickhouse-server.log**

If you see your changes in **preprocessed\_configs** it does not mean that changes are applied on running server, check [Settings & restart](altinity-kb-server-config-files.md#Settings-%26--restart)

© 2021 Altinity Inc. All rights reserved.

