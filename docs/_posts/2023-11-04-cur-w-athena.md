---
layout: post
title: "Querying AWS Cost & Usage Reports with Athena"
---

Athena is one of those AWS services I wish I found myself needing more often. It's such a nice tool. It's fairly intuitive (as far as data engineering tools go), very cheap to use, and its serverless nature makes the bar of entry for experimentation extremely low.

I recently covered how to generate Cost & Usage Reports ("CUR") at scale [in another post](/2023/10/31/non-org-cur.html). Now it's time to try to make something useful with those CUR. If you're impatient and just want the code, you can [check the Github repo with the CUR generation and the SQL queries instead](https://github.com/otsu81/aws-cur-athena).

I'm going to assume that a number of CUR are aggregated and stored in an S3 bucket according to the pattern

```
s3://s3://{bucketname}/{someprefix}/account={accountId}
```

CUR will automatically save its files with the prefix `{granularity}`, e.g. if the chosen granularity is `DAILY` a file might have the complete S3 path

```
s3://cost-usage-reports-aggregated/cur/account=123456789012/year=2023/month=10/123456789012-cur-00001.snappy.parquet
```

The reason for this structure is simple - any part of the object's key that contains the string `{key}={variable}` can be used as a partition when loading the bucket's contents into Athena's database. [Read more about partitioning in Athena.](https://docs.aws.amazon.com/athena/latest/ug/partitions.html)

## Why Athena and not *[something else]* for this?

Our original problem was to figure out what the actual cost of some services were across all of the AWS accounts we have in a simple fashion. We don't have access to neither AWS Organizations nor the main payee account and are forced to use other means to aggregate our costs.

We looked at other solutions, including QuickSight, but quickly realized it would either be very costly to use or complicated to set up. Not all tools that claim that they are able to cover the usecase are very good at it either.

Athena presents a reasonable middleground by allowing us to query data stored in S3 directly, and with SQL (which many of us already know) we can create incredibly powerful queries that run in seconds. You only pay for the data stored in S3, the data transfer from S3 when loading the Athena database, and the runtime of the queries. Compared to the alternatives the costs are literally peanuts.

## A few words about CUR

The CUR is incredibly detailed, with many columns you might not necessarily need. It can be meaningful to go through the columns to identify which ones might not contain information you're interested in. The fewer columns the faster (and simpler) your queries will be. The table suggested by AWS contains a lot of unnecessary rows that we will most likely never use. For example, you will most likely [not need to use the `blended_cost`](https://aws.amazon.com/blogs/aws-cloud-financial-management/understanding-your-aws-cost-datasets-a-cheat-sheet/) columns.

## Generating the Athena table

Before generating the table, make sure you have [created your Athena database](https://docs.aws.amazon.com/athena/latest/ug/creating-databases.html) to identify which bucket and which object prefixes are going to be ingested.

An example query using a subset of some of the more interesting columns available:

```sql
CREATE EXTERNAL TABLE CostUsageReportForAthena.cost_usage_report_for_athena(
  identity_line_item_id STRING,
  identity_time_interval STRING,
  bill_payer_account_id STRING,
  bill_billing_period_start_date TIMESTAMP,
  bill_billing_period_end_date TIMESTAMP,
  line_item_usage_account_id STRING,
  line_item_usage_start_date TIMESTAMP,
  line_item_usage_end_date TIMESTAMP,
  line_item_product_code STRING,
  line_item_line_item_type STRING,
  line_item_usage_type STRING,
  line_item_currency_code STRING,
  line_item_unblended_rate STRING,
  line_item_unblended_cost DOUBLE,
  line_item_line_item_description STRING,
  product_description STRING,
  product_free_overage STRING,
  product_free_query_types STRING,
  product_free_tier STRING,
  product_free_usage_included STRING,
  product_frequency_mode STRING,
  product_from_location STRING,
  product_from_location_type STRING,
  product_from_region_code STRING,
  product_instance_family STRING,
  product_instance_name STRING,
  product_instance_type STRING,
  product_instance_type_family STRING,
  product_invocation STRING,
  product_product_family STRING,
  product_region STRING,
  product_servicecode STRING,
  product_servicename STRING,
  product_sku STRING
)

PARTITIONED BY (
  account STRING,
  year STRING,
  month STRING
)

ROW FORMAT SERDE 'org.apache.hadoop.hive.ql.io.parquet.serde.ParquetHiveSerDe'
WITH  SERDEPROPERTIES (
  'serialization.format' = '1'
) LOCATION 's3://cost-usage-reports-aggregated/curs/'
```

## Examples for querying CUR:

We can start off by seeing which is the most expensive account we have, from most to least expensive:

```sql
SELECT account, sum(line_item_unblended_cost) AS total_cost
FROM cost_usage_report_for_athena
GROUP BY account
ORDER BY total_cost DESC
```

To find out what the total cost is per account per month:

```sql
SELECT line_item_product_code, sum(line_item_blended_cost) AS cost, month
FROM cost_usage_report_for_athena
GROUP BY line_item_product_code, month
HAVING sum(line_item_blended_cost) > 0
ORDER BY cost DESC;
```

Let's see which services are the most expensive, and how much we've spent on them so far:

```sql
SELECT line_item_product_code, sum(line_item_blended_cost) AS cost, month
FROM cost_usage_report_for_athena
GROUP BY line_item_product_code, month
HAVING sum(line_item_blended_cost) > 0
ORDER BY cost DESC;
```

The central cloud team in my organization deploys a number of security services that are non-optional. Let's find out what the total percentage of or cloud bill is related to the security services pushed out by my particular org's organization:

```sql
SELECT
  (
    SELECT SUM(line_item_unblended_cost)
    FROM cost_usage_report_for_athena
    WHERE product_servicecode IN ('AWSConfig', 'AWSSecurityHub', 'AmazonInspectorV2', 'AmazonGuardDuty')
  )
  /
  (
    SELECT SUM(line_item_unblended_cost)
    FROM cost_usage_report_for_athena
  )
  * 100 AS percentage_cost;
```

I wonder if we can detect any days that had unusual cost spikes?

```sql
WITH DailyCost AS (
  SELECT DATE(line_item_usage_start_date) AS usage_date, account,
    SUM(line_item_unblended_cost) AS daily_cost
  FROM cost_usage_report_for_athena
  WHERE line_item_line_item_type != 'Tax'
  GROUP BY DATE(line_item_usage_start_date), account
),
AverageCost AS (
  SELECT account, AVG(daily_cost) AS avg_daily_cost
  FROM DailyCost
  GROUP BY account
)
SELECT d.usage_date, d.account, d.daily_cost
FROM DailyCost d
JOIN AverageCost a ON d.account = a.account
WHERE d.daily_cost > a.avg_daily_cost
ORDER BY d.daily_cost DESC;
```

These example queries are just the tip of the proverbial iceberg. We find new ways of making queries all the time. If you have a well-structured dataset I can highly recommend giving Athena a try.