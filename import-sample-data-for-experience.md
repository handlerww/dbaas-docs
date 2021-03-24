---
title: Import Sample Data
summary: TBD
---

# Import Sample Data

This document describe how to import sample data via UI. Examples used in the TiDB manual use System Data from Capital Bikeshare, released under the Capital Bikeshare Data License Agreement. In this document, you need to have one TiDB cluster for operations below.

1. Click **import** button at the right top of the block of the cluster you want to import data into. You will get into the **Data Import Task** page.

2. At the data import task page, choose **AWS S3** at **Data Source Type** row.

3. Fill the blank **Source URL** with the sample data url `s3://yiwen-pp/data-ingesting-sample-data/` (sample data URL is TBD).

4. Select **US West(Oregon)** at **Region**.

5. Choose **Aurora Backup Snapshot** at **Data Format**.

6. Fill the blank in **Target Database** with your TiDB cluster access information.

7. Leave **DB/Tables Filter** in blank.

8. Click **import** button, the data ingestion process is on going

Once the cluster finished the Importing progress, you will get the sample data in your database.
