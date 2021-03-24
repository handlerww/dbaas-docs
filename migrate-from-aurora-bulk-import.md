---
title: Migrate from Amazon Aurora MySQL to TiDB Cloud in Bulk
summary: Learn how to migrate data from Amazon Aurora MySQL to TiDB Cloud in bulk.
---

# Migrate from Amazon Aurora MySQL to TiDB in Bulk

This document describes how to migrate data from Amazon Aurora MySQL to TiDB Cloud in bulk using the import tools on TiDB Cloud console. 

> **Note:**
>
> It's a tutorial for the usage of Import UI on TiDB Cloud consle. It's recommended to use UI first before you reading this document.

## Learn how to create an import task on the TiDB Cloud console

Currently, you can only use the import tools on TiDB Cloud console on the tier higher than `T1.tiny`. To import data, perform the following steps:

1. On the TiDB Cloud console, click **Import** on the top of the cluster block. 
2. Prepare source data according to [Learn how to create an Amazon S3 Bucket and prepare source data files](#learn-how-to-create-an-amazon-s3-bucket-and-prepare-source-data-files). You can see the advantages and disadvantages of different **Data Format** in the preparing data part.
3. Fill in the **Data Source Type**, **Bucket Name**, **Region**, and **Data Format** fields according to the specification of your source data.
4. Fill in the **Username** and **Password** fields of the **Target Database** according to the connection settings of your cluster.
5. Create the bucket policy and role for cross-account access according to [Learn how to create a Policy and Role for Cross-Account access](#learn-how-to-create-a-policy-and-role-for-cross-account-access).
6. Click **Import** to create the task.

> **Note:**
>
> If your task fails, refer to [Learn how to clean up incomplete data](#learn-how-to-clean-up-incomplete-data).

## Learn how to create an Amazon S3 Bucket and prepare source data files

To prepare data, you can select one from the following two options:

- [Option 1: Prepare source data files using Dumpling](#option-1-prepare-source-data-files-using-dumpling)

    You need to launch [Dumpling](https://docs.pingcap.com/tidb/stable/dumpling-overview) on your EC2, and export the data to Amazon S3. The data you export is the current latest data of your source database. This might affect the online service. Dumpling will lock the table when you export data.

- [Option 2: Prepare source data files using Amazon Aurora snapshots](#option-2-prepare-source-data-files-using-amazon-aurora-snapshots)

    This affects your online service. It might take a while when you export data, because the export task on Amazon Aurora first restores and scales the database before exporting data to Amazon S3. For more details, see [Exporting DB snapshot data to Amazon S3](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/USER_ExportSnapshot.html).

### Prechecks and preparations

> **Note:**
>
> Currently, it is not recommended to import more than 2 TB of data.
>
> Before starting the migration, you need to do the following prechecks and preparations.

#### Ensure enough free space

Ensure that the free space of your TiDB cluster is larger than the size of your data. It is recommended that you should reserve 600 GB free space on each TiKV node. You can add more TiKV nodes to fulfill your demand.

#### Check the database’s collation set settings

Currently, TiDB only supports the `utf8_general_ci` and `utf8mb4_general_ci` collation. To verify the collation settings of your database, execute the following command in the MySQL terminal connected to Aurora:

{{< copyable "sql" >}}

```sql
select * from ((select table_schema, table_name, column_name, collation_name from information_schema.columns where character_set_name is not null) union all (select table_schema, table_name, null, table_collation from information_schema.tables)) x where table_schema not in ('performance_schema', 'mysql', 'information_schema') and collation_name not in ('utf8_bin', 'utf8mb4_bin', 'ascii_bin', 'latin1_bin', 'binary', 'utf8_general_ci', 'utf8mb4_general_ci');
```

The result is as follows:

```output
Empty set (0.04 sec)
```

If TiDB does not support your character set or collation, consider converting them to supported types. For more details, see [Character Set and Collation](https://docs.pingcap.com/tidb/stable/character-set-and-collation).

### Option 1: Prepare source data files using Dumpling

You need to prepare an EC2 to run the following data export task. It's better to run on the same network with Aurora and S3 to avoid extra fees.

1. Download Dumpling on EC2.

    Download the TiDB Toolkit:

    {{< copyable "shell-regular" >}}

    ```bash
    mkdir tidb-toolkit-latest-linux-amd64 && \
    wget -qO- https://download.pingcap.org/tidb-toolkit-latest-linux-amd64.tar.gz|tar -xzv -C tidb-toolkit-latest-linux-amd64 --strip-components 1
    ```

2. Grant the write privilege to Dumpling for writing S3.

    > **Note:**
    >
    > If you have assigned the IAM role to the EC2, you can skip configuring the access key and security key, and directly run Dumpling on this EC2.
    
    You can grant the write privilege using the access key and security key of your AWS account in the environment. Create a specific key pair for preparing data, and revoke the access key immediately after you finish the preparation.

    {{< copyable "shell-regular" >}}

    ```bash
    export AWS_ACCESS_KEY_ID=AccessKeyID
    export AWS_SECRET_ACCESS_KEY=SecretKey
    ```

3. Back up the source database to S3.

    Use Dumpling to export the data from Amazon Aurora. Based on your environment, replace the content in angle brackets (>), and then execute the following commands. If you want to use filter rules when exporting the data, refer to [Table Filter](https://docs.pingcap.com/tidb/stable/table-filter#cli).

    {{< copyable "shell-regular" >}}

    ```bash
    export_username="<Aurora username>"
    export_password="<Aurora password>"
    export_endpoint="<the endpoint for Amazon Aurora MySQL>"
    # You will use the s3 url when you create importing task
    backup_dir="s3://<bucket name>/<backup dir>"
    s3_bucket_region="<bueckt_region>"

    ./tidb-toolkit-latest-linux-amd64/bin/dumpling \
    -u "$export_username" \
    -p "$export_password" \
    -P 3306 \
    -h "$export_endpoint" \
    --filetype sql \
    --threads 8 \
    -o "$backup_dir" \
    --consistency="none" \
    --s3.region="$s3_bucket_region" \
    -r 200000 \
    -F 256MiB
    ```

4. On the data import task panel of TiDB Cloud, choose **TiDB Dumpling** as the **Data Format**.

### Option 2: Prepare source data files using Amazon Aurora snapshots

#### Back up the schema of the database and restore on TiDB Cloud

To migrate data from Aurora, you need to back up the schema of the database.

1. Install the MySQL client.

    {{< copyable "sql" >}}

    ```bash
    yum install mysql -y
    ```

2. Back up the schema of the database.

    {{< copyable "sql" >}}

    ```bash
    export_username="<Aurora username>"
    export_endpoint="<Aurora endpoint>"
    export_database="<Database to export>"

    mysqldump -h ${export_endpoint} -u ${export_username} -p --ssl-mode=DISABLED -d${export_database} >db.sql
    ```

3. Import the schema of the database into TiDB Cloud. 

    {{< copyable "sql" >}}

    ```bash
    dest_database="<TiDB Cloud connect endpoint>"
    dest_username="<TiDB Cloud username>"
    dest_database="<Database to restore>"

    mysql -u ${dest_username} -h ${dest_database} -P 4000 -p -D${dest_database}<db.sql
    ```

4. On the data import task panel of TiDB Cloud, choose **Aurora Backup Snapshot** as the **Data Format**.

#### Take a snapshot and export it to S3

 1. From the Amazon RDS console, choose **Snapshots**, and click **Take snapshots** to create a manual snapshot.

 2. Fill in the blank under **Snapshot Name**. Click **Take snapshot**. When you finish creating the snapshot, the snapshot shows under the snapshot table.

 3. Choose the snapshot you have taken, click **Actions**. In the drop-down box, click **Export to Amazon S3**.

 4. Fill the blank under **Export identifier**.

 5. Choose the amount of data to be exported. In this guide, **All** is selected. You can also choose partial to use identifiers to decide which part of the database needs to be exported.

 6. Choose the S3 bucket to store the snapshot. You can create a new bucket to store the data for security concerns. It is recommended to use the bucket in the same region as your TiDB cluster. Downloading data across regions can cause additional network costs.

 7. Choose the proper IAM role to grant write access to S3 bucket. Now you can see the progress in the task table.

 8. From the task table, record the destination bucket (for example, s3://snapshot-bucket/snapshot-samples-1), fill it in the blank named **Bucket Name**.

 9. Modify the encryption key type of server-side encryption to Amazon S3 Key.

     Choose the backup directory, and click **Actions**. In the pop-up list, click to edit server-side encryption.

     Choose **Amazon S3 key (SSE-S3)**. It might take a while, depending on your database type and size.

## Learn how to create a Policy and Role for Cross-Account access

For security concerns, you need to create a policy and role for cross-account access. You can use either of the following two options to apply the bucket policy for cross-account access:

- [Option 1: Apply the bucket policy for cross-account access using AWS console](#option-1-apply-the-bucket-policy-for-cross-account-access-using-aws-console)
- [Option 2: Apply the bucket policy for cross-account access using awscli](#option-2-apply-the-bucket-policy-for-cross-account-access-using-awscli)

### Option 1: Apply the bucket policy for cross-account access using AWS console

1. Sign in to the AWS console and go to **S3 Manage Console**.
2. Choose the bucket that stores the data to import.
3. On the top panel, select **Permissions**.
4. Click **Edit** to edit **Bucket policy**.
5. Paste the policy data from the TiDB Cloud **Import Task Panel**, and apply the changes. In this permission policy, **GetObject** and **ListBucket** permissions are required. If bucket policies exist, append the policies in **Statement** to your bucket policies list.

    ![Get bucket policy example](media/migrate-from-aurora-bulk-import/guide-1.png)

Once finished, you have created a policy and role for cross-account. You can continue with the configuration on the data import task panel of TiDB Cloud.

### Option 2: Apply the bucket policy for cross-account access using awscli

1. Install awscli and configure awscli according to your account information. You can refer to [AWS cli configure quickstart](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html) to see how to get the information needed by awscli.

    {{< copyable "shell-regular" >}}

    ```bash
    curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
    unzip awscliv2.zip
    sudo ./aws/install
    # Configure the awscli with your account information
    aws configure
    ```

2. Export the data from the dashboard to the local file `policy.json`.

3. Execute the command from the task dashboard to apply the bucket policy. You can just copy the command on the TiDB Cloud console.

    ```
    aws s3api put-bucket-policy --bucket ${bucket_name} --policy policy.json
    ```

> **Note:**
>
> To have the ListObject permission, you must get the ListBucket permission first, which is required by Amazon S3. For more details, see this [answer](https://aws.amazon.com/premiumsupport/knowledge-center/s3-access-denied-listobjects-sync/) on AWS.

## Learn how to set up filter rules

Refer to the [Table Filter](https://docs.pingcap.com/tidb/stable/table-filter#cli) document. Currently, TiDB Cloud only supports one table filter rule.

## Learn how to clean up incomplete data

You can check the requirements again. When all the problems are solved, you can drop the incomplete database and restart the importing process.
