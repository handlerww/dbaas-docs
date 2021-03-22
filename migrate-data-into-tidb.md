---
title: Migrate Data into TiDB
summary: This page has instructions for migrating data from MySQL compatible databases to TiDB using the Dumpling and TiDB Lightning tools.
---

# Migrate Data into TiDB

This document describes how to migrate your data from MySQL compatible databases to [TiDB Cloud](https://pingcap.com/products/tidbcloud) using [Dumpling](https://docs.pingcap.com/tidb/stable/dumpling-overview) and [TiDB Lightning](https://docs.pingcap.com/tidb/stable/tidb-lightning-overview) tools. Dumpling exports data from any MySQL-compatible database, and TiDB Lightning imports it into TiDB. You can smoothly migrate data to TiDB from any MySQL-compatible database. 

To migrate data, do the following: 

1. Make sure your environment meets the [migration prerequisites](#migration-prerequisites).
2. [Prepare your working environment](#prepare-the-working-environment-for-data-migration).
3. Use Dumpling to export the data from the source database.
4. Use TiDB to import the data to TiDB Cloud.

## Migration prerequisites

### MySQL compatible database requirements

Confirm whether TiDB supports the collations of the tables you want to migrate. The default character set in TiDB is `utf8mb4`, which matches the default in MySQL 8.0 and above. TiDB differs from MySQL and defaults to using a binary collation. This binary collation uses a case-insensitive collation. Currently, TiDB supports the following character sets:

* `utf8`
* `utf8mb4`
* `ASCII`
* `latin1`
* `binary`

TiDB supports the following collations:

* `ascii_bin`
* `binary`
* `latin1_bin`
* `utf8mb4_general_bin`
* `utf8_general_bin`

You can learn more about [Character Sets and Collations](https://docs.pingcap.com/tidb/stable/character-set-and-collation) in the TiDB documentation. 

### Working environment to run tools for migration

You will be running the migration tools in an local instance. Make sure the instance meets the following requirements:

* Network access
    * Your instance can access both the source database and the database on TiDB Cloud.
    * Set up VPC peering between TiDB Cloud and the VPC of your instance. For details, refer to [Set Up VPC Peering Connections](https://docs.pingcap.com/tidbcloud/beta/set-up-vpc-peering-connections).
* Storage
    * Based on the size of your database, specify the storage capacity. The backup files are temporarily stored on a storage volume. The free disk of the volume should be larger than the size of the database.
* Computing resources
    * You'll do all the operations described in this document on this machine. If you need to migrate large quantities of data and you are concerned about performance, you may choose a better-performing instance.

## Migrate data into TiDB Cloud

### Export data from the source database

The TiDB Toolkit package includes Dumpling and TiDB Lighting.

1. Download the TiDB Toolkit using the following commands. 

    {{< copyable "shell-regular" >}}

    ```shell
    mkdir tidb-toolkit-latest-linux-amd64 && \
    wget -qO- https://download.pingcap.org/tidb-toolkit-latest-linux-amd64.tar.gz|tar -xzv -C tidb-toolkit-latest-linux-amd64 --strip-components 1
    ```

2. Use Dumpling to export the data. Based on your environment, replace the content in angle brackets (&lt;&gt;), and then execute the following commands.

    {{< copyable "shell-regular" >}}

    ```shell
    export_username=<Source database username>
    export_password=<Source database password>
    export_endpoint=<the endpoint for the Source database>
    backup_dir=<backup directory>

    ./tidb-toolkit-latest-linux-amd64/bin/dumpling \
      -u "$export_username" \
      -p "$export_password" \
      -P 3306 \
      -h "$export_endpoint" \
      --filetype sql \
      --threads 8 \
      -o "$backup_dir" \
      -f "*.*" \
      --consistency="none" \
      -F 256MiB
    ```

### Import data into TiDB Cloud

To import data into TiDB Cloud, replace the content included in angle brackets based on your TiDB Cloud cluster settings, and execute the following commands. If the size of your data is too large, you could use `tmux` or `nohup` to keep the TiDB Lightning process up.

For security purposes, the TiKV backend isn't exposed to the customer. For now, we use TiDB Lightning's TiDB-backend mode to import data.

{{< copyable "shell-regular" >}}

```shell
backup_dir=<backup directory>
tidb_endpoint=<endpoint of the cluster in TiDB Cloud>
tidb_username=<TiDB username>
tidb_password=<TiDB password>
tidb_port=<TiDB port>

./tidb-toolkit-latest-linux-amd64/bin/tidb-lightning --backend tidb -check-requirements=false \
-d=$backup_dir \
-server-mode=false \
-tidb-host="$tidb_endpoint" \
-tidb-port="$tidb_port" \
-tidb-user="$tidb_username" \
-tidb-password="$tidb_password"
```

> **Note:**
>
> If you manually interrupt TiDB Lightning, a checkpoint will be created to help to identify where to resume the restore process. If you encounter any problems that cannot be fixed automatically, refer to [TiDB Lightning Checkpoints](https://docs.pingcap.com/tidb/stable/tidb-lightning-checkpoints).

When the `tidb-lightning` process completes, The TiDB cluster on TiDB Cloud will have the same structure and data as the source cluster.
