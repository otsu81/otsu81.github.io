---
layout: post
title: "Aggregated AWS Cost & Usage Reports"
---

I'm part of a team having complete autonomy of around a dozen or so AWS accounts. We are one of many projects/teams governed by the central IT's cloud team.

The team is responsible for all costs and budgeting for said accounts. We have a strong incentive to stay cost efficient as we get chargeback from central IT.

For those without access to the main payee account or AWS Organizations, getting an aggregated view to identify cost anomalies and spikes can be a bit tricky. There are several cost management services out there, and one could also set up QuickSight. They are however usually quite expensive and sometimes require a significant investment in time to learn.

We decided to try to leverage the [Cost & Usage Reports](https://docs.aws.amazon.com/cur/latest/userguide/what-is-cur.html) ("CUR") service from AWS instead. It's a built-in capability which generates files that can be programmatically accessed. The granularity is from 1 hour - 1 month. You can also decide what type of file you'd like generated, e.g. parquet or gzip:ed csv files.

Today I'm sharing an architecture which generates CUR and forwards them automatically to an aggregation bucket as they are produced. This makes it simple to use Athena in an efficient manner. I wrote another post about [how we use Athena on our CUR](/2023/11/04/cur-w-athena.html).

## Some thoughts about Cost & Usage Reports
CUR has lots of uses, and the fact that it's free makes it perfect to make experiments on. The only cost incurred is the writes and storage in S3 which is extremely cheap (at least in this context).

That said, I have two recommendations when generating CUR.
1. Given that CURs are produced multiple times daily and each encompasses the full history of the current month, you could end up with numerous redundant files by month-end. Overwriting eliminates unnecessary duplication unless a detailed record of AWS's adjustments is required.
1. Select daily granularity to mitigate file size, reducing both storage costs and processing complexity.

## Deploying with StackSets, and preparations
The idea is to use multiple Cloudformation deployments with [Cloudformation StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html) to simplify the creation of CUR, and aggregate it into a single bucket in the account where we will collect CUR and run our data.

Let's call the aggregation account "governance account", and the accounts generating CUR "child accounts".

![Cloudformation setup for cost and usage](/images/cfn_cur.png)

Unless manually specified, by default StackSets requires two IAM roles:
* The governance account must have the role `AWSCloudFormationStackSetAdministrationRole` which has the `sts:AssumeRole` permission
* The child accounts must have the role `AWSCloudFormationStackSetExecutionRole`, with a trust to the governance account and the Cloudformation service

You can read more about the role requirements in the [StackSets documentation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs-self-managed.html).

## Aggregation CUR bucket
Our architecture requires an S3 bucket in the governance account with a bucket policy, which allows the child accounts to put the CUR. If we're assuming our bucket name is `cost-usage-reports`, and that we will want to leverage [Athena Partitions](https://docs.aws.amazon.com/athena/latest/ug/partitions.html) for `account`, `year`, and `month`, we could have the following example bucket policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::cost-usage-reports-aggregated/curs/account=111111111111/*"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::222222222222:root"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::homesmart-cost-usage-reports-aggregated/curs/account=222222222222/*"
    },
  ]
}
```

## Cloudformation template
Because CUR can only be generated in the `us-east-1` (N. Virginia) region, we are assuming that all deployments of the Cloudformation template will be done in `us-east-1`. It's a good idea to keep everything in the same region (including the aggregation bucket and later the Athena queries) to avoid cross-region data transfer cost and latency.

Our template will create the following resources:

* An S3 bucket which will be the target of the CUR, with a 2 year lifecycle policy
* A bucket policy which will allow the AWS CUR service to write to the bucket
* A report definition which specifies the granularity to daily, overwrite existing reports (to minimize the number of files and make management simpler), and the generated file to be of parquet format
* An IAM role for Lambda which will be responsible for sending generated CUR files to the governance account's aggregation bucket
* The Lambda which will copy CUR files to the aggregation bucket
* A permission for S3 to invoke the Lambda function

[Link to cloudformation template on Github](https://github.com/otsu81/aws-cur-athena/blob/main/template.yaml)

TODO: Make bucket and policy optional by adding boolean parameter for creation of bucket policy
