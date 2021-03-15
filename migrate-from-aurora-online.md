---
title: Migrate from Amazon Aurora to TiDB Cloud online with DM
summary: Learn how to migrate data from Amazon Aurora to TiDB Cloud online with DM.
---

# Migrate from Amazon Aurora to TiDB Cloud online with DM

This guide provides step by step instructions to migrate data from Amazon Aurora to TiDB Cloud using DM.

## Prechecks and Preparations

Before you start the migration, you need to do some prechecks and some preparations to ensure a smooth migration.

### Make a plan for DM cluster topology

In this guide, you'll use 1 DM master and 1 DM worker deployed locally for functional verification.

For production, you may need to deploy more nodes for DM masters and workers, see "[To migrate more than one database](#to-migrate-more-than-one-database)" in troubleshooting.

### Enable binary logging(binlog) for Amazon Aurora

To migrate data from Amazon Aurora, you need to enable binary logging(binlog) for Amazon Aurora. If you are using the default parameter group, it's immutable, you need to refer to the 

[troubleshooting](#troubleshooting) to create a parameter group for configuring the database. If you have already created, follow these steps:

1. Open the [Amazon Relational Database Service (Amazon RDS) console](https://console.aws.amazon.com/rds). In the Aurora dashboard, select the row about the source cluster, choose "Configuration", see "DB cluster parameter group", click it.

2. Select the DB custom cluster parameter group, choose Parameter group actions, and select Edit. Change the value for binlog_format to 'ROW', Choose Save changes.

3. After you perform the changes, you must reboot the write instances to make the changes take effect.

For more information on enabling binary logging on Aurora, you could refer to [https://aws.amazon.com/premiumsupport/knowledge-center/enable-binary-logging-aurora](https://aws.amazon.com/premiumsupport/knowledge-center/enable-binary-logging-aurora)

### Check the connectivity to the instances on Amazon Aurora and TiDB Cloud

You need to prepare an EC2 for the migration jobs. The EC2 should ensure the connectivity to Amazon Aurora and TiDB Cloud. If you are not familiar with how to prepare an EC2 for migration, you could refer to [troubleshooting](#launch-an-ec2-for-migration).

For Aurora, you will use the endpoint that Amazon Aurora provides. Get the access information from the Aurora MySQL Connectivity & security page.

Display your Amazon Aurora access information

For TiDB Cloud, you need to create a cluster first. Then you need to create a VPC peering between your EC2 instance and TiDB Cloud. You could refer to [Set up VPC Peering Connections](https://docs.pingcap.com/tidbcloud/beta/set-up-vpc-peering-connections) for the information about VPC Peering. For more details about how to connect to TiDB Cloud, you could also refer to TiDB Cloud documentation [Connect to Your TiDB Cluster](https://docs.pingcap.com/tidbcloud/beta/connect-to-tidb-cluster#connect-via-a-sql-client).

### Extend the binlog retention hours

For incremental migration, it needs to extend the binlog retention hours to keep the binlog for migration.

Connect to Aurora MySQL, use the function below.

```
sql > call mysql.rds_set_configuration('binlog retention hours', 168);
```

If downstream doesn't catch up with upstream, this binlog retention hours should be extended.

And downstream almost catches up with upstream, binlog retention hours could set to a lower value to reduce disk usage.

### Specify the volume of the storage larger than the size of the data 

DM-worker stores full data in the dump and load phases. Therefore, the disk space for DM-worker needs to be greater than the total amount of current data to be migrated. If the relay log is enabled for the replication task, DM-worker needs additional disk space to store upstream binlog data.

The size of the storage should be larger than the size of your source database. Get the amount of used space on the volume from the Amazon Aurora dashboard. If you do not have enough free space, the data export may fail. 

Display the amount of used space on the volume

### Check the database's collation set settings

Currently, TiDB only supports the `utf8_general_ci` and `utf8mb4_general_ci` collation. For your convenience, we need to verify the collation settings of the database. You could execute these commands in the MySQL terminal to Aurora. 

```
mysql > select * from ((select table_schema, table_name, column_name, collation_name from information_schema.columns where character_set_name is not null) union all (select table_schema, table_name, null, table_collation from information_schema.tables)) x where table_schema not in ('performance_schema', 'mysql', 'information_schema') and collation_name not in ('utf8_bin', 'utf8mb4_bin', 'ascii_bin', 'latin1_bin', 'binary', 'utf8_general_ci', 'utf8mb4_general_ci');
```

The results show the following:

```
Empty set (0.04 sec)
```

If TiDB does not support your character set or collection, you could consider converting them to supported types. You could refer to [Character Set and Collation](https://docs.pingcap.com/tidb/stable/character-set-and-collation) for more details.

### Prepare a key pair for deployment

For TiUP, you need to provide authentication credentials for deployment.

Generate a ssh keypair:

```
ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa
```

If you do a local deployment, you could write the public key to `~/.ssh/authorized_keys`.

```
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

If you deploy on multiple nodes, you need to append the public key to the tail of `~/.ssh/authorized_keys` on every node.

When you finish the change, you will find you could connect each node without authentication.

### Disable SELinux

To enable the DM service, you need to disable selinux.

```
sudo setenforce 0
sudo sed -i --follow-symlinks 's/^SELINUX=.*/SELINUX=disabled/g' /etc/sysconfig/selinux && cat /etc/sysconfig/selinux
```

## Install TiUP for DM and deploy a DM cluster

DM can be deployed in multiple ways. Currently, it is recommended to use TiUP to deploy a DM cluster. If you want to deploy DM using binary, you could refer to [troubleshooting](#deploy-using-binary) about this. 

```
curl --proto '=https' --tlsv1.2 -sSf https://tiup-mirrors.pingcap.com/install.sh | sh
source ~/.bash_profile
tiup install dm
```

To deploy the DM cluster, you could use a password or credential for authentication. In the preparation section, you have created a key pair for authentication. 

For functional verification, you deploy one master and one worker, you could go to troubleshooting for [migrating more than one database](#to-migrate-more-than-one-database). 

```
cat > topology.yaml<<EOF
---
# Specify the control node information
global:
  user: "ec2-user"
  ssh_port: 22
  deploy_dir: "/home/ec2-user/dm/deploy"
  data_dir: "/home/ec2-user/dm/data"
 
master_servers:
  - host: 127.0.0.1
 
worker_servers:
  - host: 127.0.0.1
EOF
 
# To use password for ssh authentication, you could use `--user root -p`, for example:
# tiup dm deploy dm-test v2.0.1 ./topology.yaml --user root -p
tiup dm deploy dm-test v2.0.1 ./topology.yaml --user ec2-user
 
# start the cluster
tiup dm start dm-test
```

## Configure the source and create migration jobs

Create a data source for migration. Replace the source database information according to your instance on Aurora in the 'from' section. By default, the GTID is disabled on Aurora, you could ignore 'enable-gtid' when you are doing migration jobs. If you have enabled this, please set 'enable-gtid' to 'true'.

For more information about GTID, you could refer to [AWS Documentation | Configuring GTID-based replication for an Aurora MySQL cluster](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/mysql-replication-gtid.html#mysql-replication-gtid.configuring-aurora).

```
cat > /tmp/source.yaml<<EOF
source-id: "aurora-replica-01"
enable-gtid: false

# Replace the source database information according to your instance on Aurora. You could refer to the format below.
from:
  host: "database-1-instance-1.cbnigarpiufl.us-west-2.rds.amazonaws.com"
  user: "admin"
  password: "cHwUvFUtGeNglEbsQX0m"
  port: 3306
EOF
 
tiup dmctl --master-addr 127.0.0.1:8261 operate-source create /tmp/source.yaml
```

Create migration jobs. Replace the target database information according to the connection information of your instance on TiDB Cloud. 

For doubt on how to set block-allow-list on tables, or in other words, how to decide which table to migrate, you could refer to [Table Filter](https://docs.pingcap.com/tidb/dev/table-filter).

```
cat > /tmp/task.yaml <<EOF
name: "migrations"
task-mode: "all"
# Replace the target database information according to your instance on TiDB Cloud. You could refer to the format below.
target-database:
  host: "tidb.6657c286.23110bc6.us-west-1.prod.aws.tidbcloud.com"
  port: 4000
  user: "root"
  password: "lkqw0d012ksds"
# Replace Ended

mysql-instances:
- source-id: "aurora-replica-01"
  block-allow-list: "global"
  mydumper-config-name: "global"

# change the do-dbs to the table you want to migrate
block-allow-list:
  global:
    do-dbs: ["tpcc"]

loaders:
  global:
    pool-size: 16

EOF

tiup dmctl --master-addr 127.0.0.1:8261 start-task /tmp/task.yaml 
```

If you want to reconfigure the task, you can stop the task and relaunch the task.

```
tiup dmctl --master-addr 127.0.0.1:8261 stop-task /tmp/task.yaml 
```

## Verification

Use `dmctl` to query the status of the migration jobs. 

```
tiup dmctl --master-addr 127.0.0.1:8261 query-status
```

If the task is running normally, the following information is returned.

```
{
    "result": true,
    "msg": "",
    "tasks": [
        {
            "taskName": "test",
            "taskStatus": "Running",
            "sources": [
                "aurora-replica-01",
            ]
        }
    ]
}
```

You can query data in the downstream, modify data in Aurora, and validate the data migrated to TiDB.

## Troubleshooting

### Launch an EC2 for migration

In this section, you'll prepare an EC2 instance to run the migration tools. You need to pay attention to two issues:

* The instance should be in the same VPC as your Amazon Aurora service. This helps you smoothly connect to Amazon Aurora.  
* Ensure that the free disk space is larger than the size of your data. 

Based on the [Amazon User Guide for Linux Instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html), the process to launch an instance includes these steps:

1. Open the Amazon EC2 console. Go to [https://console.aws.amazon.com/ec2/](https://console.aws.amazon.com/ec2/).

2. Choose an Amazon Machine Image (AMI). In this procedure, use the AMI named **Amazon Linux 2 AMI (HVM), SSD Volume Type**.

3. On the **Choose an Instance Type** page. Choose t2.large.

4. Configure the instance details. Based on the network details of your Amazon Aurora service, choose the network and subnet.

5. On Step 4, add storage, you need to set the size of the volume. The size of the storage should be larger than the size of your source database. The Amazon Aurora dashboard displays the amount of used space on the volume. 

    Display the amount of used space on the volume

    Based on this information, specify the total size of your storage. If you do not have enough free space, the data export may fail.

6. Click review and launch, create a new key pair, save the key pair generated by AWS or use the existing key pair, and click '**Launch Instances**'.

For more details on how to launch an EC2 instance, see [AWS EC2 Get Started Guide](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html).

### To migrate more than one database

From the architecture of DM, you could see that DM-worker is registered as a following node of the Aurora instance. It means if you have more than one Aurora instance to migrate, you need to keep that DM-workers are more than or equal to the source instances.

Sometimes you have more than one database to migrate, you need to configure the deployment configuration and the source migration. Here are some configuration files for you.

To deploy the cluster, set more than one master and worker node like this. Meanwhile, some monitoring servers are usually needed but not necessary.

In the section **Prepare a key pair for deployment**, you have generated a key pair for authentication. To deploy on multiple nodes, you need to copy the public key to every node listed in the topology configuration.

Show the content of the public key by the command below:

```
cat ~/.ssh/id_rsa.pub
```

Append the content in the public key to other nodes.

```
vi ~/.ssh/authorized_keys
```

When you finished the changes, deploy with TiUP.

```
cat > topology.yaml<<EOF
---
global:
  user: "ec2-user"
  ssh_port: 22
  deploy_dir: "/home/ec2-user/dm/deploy"
  data_dir: "/home/ec2-user/dm/data"
  # arch: "amd64"

master_servers:
  - host: 172.19.0.101
  - host: 172.19.0.102
  - host: 172.19.0.103

worker_servers:
  - host: 172.19.0.101
  - host: 172.19.0.102
  - host: 172.19.0.103

monitoring_servers:
  - host: 172.19.0.101

grafana_servers:
  - host: 172.19.0.101

alertmanager_servers:
  - host: 172.19.0.101
EOF
 
# You could use `--user root -p` for ssh authentication with password.
tiup dm deploy dm-test v2.0.1 ./topology.yaml --user ec2-user -i /home/ec2-user/.ssh/tiup_private

# start the cluster
tiup dm start dm-test
```

To configure the source, using 'tiup dmctl' to connect to one of the DM masters. You could deploy more than one source like this.

```
# Aurora-1
# Replace the source database information according to your instance on Aurora. You could refer to the format below.
cat > /tmp/source01.yaml<<EOF
source-id: "aurora-replica-01"
enable-gtid: false
from:
  host: "test-dm-2-0.cluster-czrtqco96yc6.us-east-2.rds.amazonaws.com"
  user: "root"
  password: "12345678"
  port: 3306
EOF
tiup dmctl --master-addr 172.19.0.101:8261 operate-source create /tmp/source01.yaml

# Aurora-2
# Replace the source database information according to your instance on Aurora. You could refer to the format below.
cat > /tmp/source02.yaml<<EOF
source-id: "aurora-replica-02"
enable-gtid: false
from:
  host: "test-dm-2-0-2.cluster-czrtqco96yc6.us-east-2.rds.amazonaws.com"
  user: "root"
  password: "12345678"
  port: 3306
EOF
tiup dmctl --master-addr 172.19.0.101:8261 operate-source create /tmp/source02.yaml
```

Create a migrate jobs. In the job configuration file here, there are more than one source in **mysql-instances** section. They share the same block-allow-list, which named **global**. You can define more block-allow-list below.

```
cat > /tmp/task.yaml<<EOF

# The task name. You need to use a different name for each of the multiple tasks that run simultaneously.
name: "test"
# The full data migration plus incremental replication task mode.
task-mode: "all"
# The downstream TiDB configuration information.
# Replace the destination database information according to your instance on TiDB Cloud. You could refer to the format below.
target-database:
  host: "tidb.6657c286.23110bc6.us-east-1.prod.aws.tidbcloud.com"
  port: 4000
  user: "root"
  password: "87654321"

# Configuration of all the upstream MySQL instances required by the current data migration task.
mysql-instances:
- source-id: "aurora-replica-01"
  # The configuration items of the block and allow lists of the schema or table to be migrated, used to quote the global block and allow lists configuration. For global configuration, see the `block-allow-list` below.
  block-allow-list: "global"
  mydumper-config-name: "global"

- source-id: "aurora-replica-02"
  block-allow-list: "global"
  mydumper-config-name: "global"

# The configuration of block and allow lists.
block-allow-list:
  global:                             # Quoted by block-allow-list: "global" above
    do-dbs: ["migrate_me"]            # The allow list of the upstream table to be migrated. Database tables that are not in the allow list will not be migrated.

loaders:
  global:
    pool-size: 16

# The configuration of the dump unit.
mydumpers:
   global:                            # Quoted by mydumper-config-name: "global" above
    extra-args: "--consistency none --statement-size 1000000 --filesize 16M"  # Aurora does not support FTWRL, you need to configure this option to bypass FTWRL.
EOF
tiup dmctl --master-addr 127.0.0.1:8261 start-task /tmp/task.yaml --remove-meta
```

### Create a parameter group to modify the configuration

Because the default parameter group is immutable, you need to modify the parameter group to configure your Aurora instance. By default, if you just created an Aurora instance, you are using the default parameter. You could check this in the details of the cluster.

The default parameter group is immutable. Therefore, you need to create a parameter group.

1. In the navigation pane, choose Parameter groups. Click 'Create parameter group'

2. Choose type 'DB Cluster Parameter Group', click 'Create' button.

3. Open the Amazon RDS console. In the navigation pane, under Clusters, choose Modify.

4. Update the DB Cluster Parameter Group to the new DB cluster parameter group, and then choose to Apply immediately.

Once the cluster parameter group is configured, you could modify the binlog feature in this parameter group. Follow the steps in the articles.

### Deploy using binary

Sometimes you may feel more familiar with the binary for deployment. The difference between deploying with TiUP and binary deployment is mainly on the deployment stage. Another difference is you need to use dmctl directly rather than 'tiup dmctl' to operate the DM cluster.   

Download the DM and deploy one dm-master and one dm-worker. 

```
curl https://download.pingcap.org/dm-v2.0.1-linux-amd64.tar.gz | tar -xzv --strip-components 1 

cat > dm-master.toml<<EOF
# DM-master1 Configuration.
name = "master"
master-addr = ":8261"
advertise-addr = "127.0.0.1:8261"
peer-urls = "127.0.0.1:8291"
initial-cluster = "master=http://127.0.0.1:8291"
EOF

./bin/dm-master --config=dm-master.toml --log-file=dm-master.log >> dm-master.log 2>&1 &

cat > dm-worker.toml<<EOF
# DM-worker1 Configuration
name = "worker"
worker-addr="0.0.0.0:8262"
advertise-addr="127.0.0.1:8262"
join = "127.0.0.1:8261"
EOF
./bin/dm-worker --config=dm-worker.toml --log-file=dm-worker.log >> dm-worker.log 2>&1 &
```

Create the datasource for Aurora and the migration tasks, use dmctl without tiup. Replace the content in the angle brackets(&lt;>).

```
cat > source.toml<<EOF
source-id: "aurora-replica-01"
enable-gtid: false

# Replace the source database information according to your instance on Aurora. You could refer to the format below.
from:
  host: "<the source endpoint of the Aurora>"
  user: "<username of Aurora>"
  password: "<password of Aurora>"
  port: 3306
EOF

./bin/dmctl --master-addr=127.0.0.1:8261 operate-source create source.toml

cat > task.yaml<<EOF
name: "migrations"
task-mode: "all"
# Replace the target database information according to your instance on TiDB Cloud. You could refer to the format below.
target-database:
  host: "<the TiDB Cloud cluster endpoint>"
  port: "<the TiDB Cloud cluster port>"
  user: "<the TiDB Cloud cluster username>"
  password: "<the TiDB Cloud cluster password>"
# Replace Ended

mysql-instances:
- source-id: "aurora-replica-01"
  block-allow-list: "global"
  mydumper-config-name: "global"

# change the do-dbs to the table you want to migrate
block-allow-list:
  global:
    do-dbs: ["<table1 to be migrated>","<table2 to be migrated>"]

loaders:
  global:
    pool-size: 16

EOF

./bin/dmctl -master-addr 127.0.0.1:8261 start-task task.yaml
```

For more information, you could see [Deploy Data Migration Using DM Binary](https://docs.pingcap.com/tidb-data-migration/v2.0/deploy-a-dm-cluster-using-binary/).
