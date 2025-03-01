---
title: "Network Configuration"
linkTitle: "Network Configuration"
description: >
    Network Configuration
---



### **Networking And Server Room Planning**

The network used for your ClickHouse cluster should be a fast network, ideally 10 Gbit. ClickHouse nodes generate a lot of traffic along with the Zookeeper connections and inter-Zookeeper communications.

For the zookeeper low latency is more important than bandwidth.

Keep the replicas isolated on the hardware level. This allows for cluster failover from possible outages.

* For Physical Environments: Avoid placing 2 ClickHouse replicas on the same server rack. Ideally, they should be on isolated network switches and an isolated power supply.
* For Clouds Environments: Use different availability zones between the ClickHouse replicas when possible \(but be aware of the interzone traffic costs\) 

These considerations are the same as the Zookeeper nodes.

For example:

| **Rack** | **Server** | **Server** | **Server** | **Server** |
| :--- | :--- | :--- | :--- | :--- |
| **Rack 1** | **CH\_SHARD1\_R1** | **CH\_SHARD2\_R1** | **CH\_SHARD3\_R1** | **ZOO\_1** |
| **Rack 2** | **CH\_SHARD1\_R2** | **CH\_SHARD2\_R2** | **CH\_SHARD3\_R2** | **ZOO\_2** |
| **Rack 3** | **ZOO3** |  |  |  |

#### **Network Ports And Firewall**

ClickHouse listens the following ports:

* 9000: clickhouse-client, native clients, other clickhouse-servers connect to here.
* 8123: HTTP clients
* 9009: Other replicas will connect here to download data.

For more information, see [CLICKHOUSE NETWORKING, PART 1](https://www.altinity.com/blog/2019/3/15/clickhouse-networking-part-1).

Zookeeper listens the following ports:

* 2181: Client connections.
* 2888: Inter-ensemble connections.
* 3888: Leader election.

Outbound traffic from ClickHouse connects to the following ports:

* ZooKeeper: On port 2181.
* Other CH nodes in the cluster: On port 9000 and 9009.
* Dictionary sources: Depending on what was configured such as HTTP, MySQL, Mongo, etc.
* Kafka or Hadoop: If those integrations were enabled.

### **SSL**

For non-trusted networks enable SSL/HTTPS. If acceptable, it is better to keep interserver communications unencrypted for performance reasons.

### **Naming Schema**

The best time to start creating a naming schema for the servers is before they’re created and configured.

There are a few features based on good server naming in ClickHouse:

* clickhouse-client prompts: Allows a different prompt for clickhouse-client per server hostname.
* Nearest hostname load balancing: For more information, see [Nearest Hostname](https://clickhouse.yandex/docs/en/operations/settings/settings/#load_balancing-nearest_hostname).

A good option is to use the following:

{datacenter}-{serverroom}-{rack identifier}-{clickhouse cluster identifier}-{shard number or server number}.

Other examples:

* rxv-olap-ch-master-sh01-r01:
  * rxv - location \(rack\#15\)
  * olap - product name
  * ch = clickhouse
  * master = stage
  * sh01 = shard 1
  * r01 = replica 1
* hetnzerde1-ch-prod-01.local:
  * hetnzerde1 - location \(also replica id\)
  * ch = clickhouse
  * prod = stage
  * 01 - server number / shard number in that DC
* sh01.ch-front.dev.aws-east1a.example.com:
  * sh01 - shard 01
  * ch-front - cluster name
  * dev = stage
  * aws = cloud provider
  * east1a = region and availability zone

#### **Host Name References**

* [**What are the best practices for domain names \(dev, staging, production\)?**](https://stackoverflow.com/a/39336460/1555175)
* [**9 Best Practices and Examples for Working with Kubernetes Labels**](https://www.replex.io/blog/9-best-practices-and-examples-for-working-with-kubernetes-labels)
* [**Thoughts On Hostname Nomenclature**](https://devcentral.f5.com/s/articles/thoughts-on-hostname-nomenclature)

### **Additional Hostname Tips**

* Hostnames configured on the server should not change. If you do need to change the host name, one reference to use is [How to Change Hostname on Ubuntu 18.04](https://linuxize.com/post/how-to-change-hostname-on-ubuntu-18-04/).
* The server should be accessible to other servers in the cluster via it’s hostname. Otherwise you will need to configure interserver\_hostname in your config.
* Ensure that `hostname --fqdn` and `getent hosts $(hostname --fqdn)` return the correct name and ip.

