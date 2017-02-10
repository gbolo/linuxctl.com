---
categories:
  - "administration"
  - "database"
keywords:
  - "memsql"
tags:
  - "memsql"
  - "database"
  - "cli"
  - "reference"
date: "2016-04-16"
title: "MemSQL - Cheatsheet"
thumbnailImagePosition: "left"
thumbnailImage: "/images/thumb/thumb-memsql.jpg"

---

[MemSQL](http://www.memsql.com/) is a cool distributed In-Memory Database which offers high performance, sharded horizontal scale-out design, High Availability (with **Enterprise** edition), and the familiar SQL syntax. It also ships with a great looking GUI that displays most of information you need to know about your cluster.
<!--more-->

However, some things are better dealt with in the cli, this page is for my reference:

### Listing status

```
# List Nodes
memsql-ops memsql-list                          
 ID       Agent Id  Process State  Cluster State  Role        Host         Port
 0669D57  A116b09   RUNNING        CONNECTED      MASTER      10.1.10.51  3306
 B98DA2F  Abefe6b   RUNNING        CONNECTED      AGGREGATOR  10.1.10.52  3306
 32BA03D  A116b09   RUNNING        CONNECTED      LEAF        10.1.10.51  3307
 9472F86  Abefe6b   RUNNING        CONNECTED      LEAF        10.1.10.52  3307
 B1776D6  Abefe6b   RUNNING        CONNECTED      LEAF        10.1.10.52  3308
 B97142B  A116b09   RUNNING        CONNECTED      LEAF        10.1.10.51  3308


# Show leaves (connect via mysql)
mysql> SHOW LEAVES;
+-------------+------+--------------------+-------------+-----------+--------+--------------------+------------------------------+
| Host        | Port | Availability_Group | Pair_Host   | Pair_Port | State  | Opened_Connections | Average_Roundtrip_Latency_ms |
+-------------+------+--------------------+-------------+-----------+--------+--------------------+------------------------------+
| 10.1.10.51 | 3307 |                  1 | 10.1.10.52 |      3308 | online |                 18 |                        0.262 |
| 10.1.10.52 | 3307 |                  1 | 10.1.10.51 |      3308 | online |                 18 |                        0.561 |
| 10.1.10.52 | 3308 |                  2 | 10.1.10.51 |      3307 | online |                 17 |                        0.465 |
| 10.1.10.51 | 3308 |                  2 | 10.1.10.52 |      3307 | online |                 17 |                        0.189 |
+-------------+------+--------------------+-------------+-----------+--------+--------------------+------------------------------+
4 rows in set (0.00 sec)

# Show partitions (select db)
mysql> SHOW PARTITIONS;                     
+---------+-------------+------+--------+--------+
| Ordinal | Host        | Port | Role   | Locked |
+---------+-------------+------+--------+--------+
|       0 | 10.1.10.52 | 3308 | Master |      0 |
|       0 | 10.1.10.51 | 3307 | Slave  |      0 |
|       1 | 10.1.10.51 | 3308 | Master |      0 |
|       1 | 10.1.10.52 | 3307 | Slave  |      0 |
|       2 | 10.1.10.52 | 3308 | Master |      0 |
|       2 | 10.1.10.51 | 3307 | Slave  |      0 |
|       3 | 10.1.10.51 | 3308 | Master |      0 |
|       3 | 10.1.10.52 | 3307 | Slave  |      0 |
|       4 | 10.1.10.52 | 3308 | Master |      0 |
|       4 | 10.1.10.51 | 3307 | Slave  |      0 |
|       5 | 10.1.10.51 | 3308 | Master |      0 |
|       5 | 10.1.10.52 | 3307 | Slave  |      0 |
|       6 | 10.1.10.52 | 3308 | Master |      0 |
|       6 | 10.1.10.51 | 3307 | Slave  |      0 |
... TRUNCATED
```

### Upgrading memsql

```
# Download the latest ops and bin from: http://download.memsql.com/releases/latest/memsqlbin_amd64.tar.gz http://www.memsql.com/download/thanks/

# Upgrade memsql
root@memsql-1 ~ $ memsql-ops memsql-upgrade --file-path /home/skroot/memsqlbin_amd64_5.0.8.tar.gz
Adding a MemSQL file
Unpacking archive to determine its version
Copying file into MemSQL Ops data directory
Registering file with MemSQL Ops
Successfully added a MemSQL file with version 5.0.8
Are you sure you want to upgrade your cluster to MemSQL version 5.0.8? This will temporarily shut down your cluster. [y/N] y
Do you want to back up the data directories in your cluster before the upgrade? If you choose no, we will use less disk space during the upgrade, but the risk of data loss will be greater. [Y/n] y
Starting MemSQL Upgrade
Currently on state OfflineClusterUpgrade.OfflineClusterUpgradeInit
Currently on state OfflineClusterUpgrade.SnapshottingDatabases
0/6 MemSQL nodes have snapshotted their databases
1/6 MemSQL nodes have snapshotted their databases
2/6 MemSQL nodes have snapshotted their databases
4/6 MemSQL nodes have snapshotted their databases
6/6 MemSQL nodes have snapshotted their databases
Currently on state OfflineClusterUpgrade.StoppingCluster
0/6 MemSQL nodes stopped
1/6 MemSQL nodes stopped
2/6 MemSQL nodes stopped
6/6 MemSQL nodes stopped
Currently on state OfflineClusterUpgrade.UpgradingCluster
0/6 MemSQL nodes have upgraded
1/6 MemSQL nodes have upgraded
3/6 MemSQL nodes have upgraded
4/6 MemSQL nodes have upgraded
5/6 MemSQL nodes have upgraded
6/6 MemSQL nodes have upgraded
Currently on state OfflineClusterUpgrade.StartingCluster
0/6 MemSQL nodes started
1/6 MemSQL nodes started
3/6 MemSQL nodes started
5/6 MemSQL nodes started
6/6 MemSQL nodes started
Currently on state OfflineClusterUpgrade.DeletingOldInstalls
0/6 MemSQL nodes have cleaned up their old installs
3/6 MemSQL nodes have cleaned up their old installs
6/6 MemSQL nodes have cleaned up their old installs
Currently on state OfflineClusterUpgrade.OfflineClusterUpgradeSuccess
Successfully upgraded cluster to MemSQL version 5.0.8

# Upgrade memsql ops
root@memsql-1 ~ $ memsql-ops agent-upgrade --file-path /home/skroot/memsql-ops-5.0.3.tar.gz              
Adding a MemSQL Ops file
Unpacking archive to determine its version
Copying file into MemSQL Ops data directory
Registering file with MemSQL Ops
Successfully added a MemSQL Ops file with version 5.0.3
Starting upgrade
Upgrading followers
Upgrading primary agent
Successfully upgraded all agents
```
