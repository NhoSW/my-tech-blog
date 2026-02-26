---
title: "S3 Table Buckets PoC: Evaluating Managed Iceberg for CDC Workloads"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [iceberg, s3, aws, trino, spark, kafka-connect, compaction, managed-service]
showTableOfContents: true
summary: "AWS S3 Table Buckets offer managed Iceberg tables with automatic compaction. We ran a PoC to see if they could solve our CDC table compaction issues. We validated Trino, Spark, and Kafka Connect integration, examined auto-compaction behavior, and assessed costs."
---

S3 Table Buckets launched in the Seoul region in March 2025. In one line: managed Iceberg tables. They provide automatic compaction, snapshot management, and reportedly higher TPS and throughput than standard S3 buckets.

Our pain point was CDC sink table compaction. Even with time partitions, non-history source tables generate updates and deletes across all partitions. Setting the compaction window to 5 or 30 days still leaves files outside that range uncompacted. Full-range compaction is impractical in terms of cost and maintenance.

Could automatic compaction solve this? We ran a PoC.

---

## Managed Service Concerns

There were genuine concerns alongside the excitement.

First, the black box problem. When issues arise in a managed service, you can't look inside. The service had just launched, with few real-world adoption reports.

Second, past experience. We had previously tried Glue/LakeFormation's managed Iceberg compaction feature. It ran as Glue-based Spark procedure invocation jobs internally, and on heavy tables it failed silently — no error logs, just no compaction.

Third, cost. Storage pricing is higher than standard buckets, and automatic compaction incurs additional charges.

---

## Glue REST Catalog Integration

S3 Table Buckets are accessed through the Glue Data Catalog's Iceberg REST endpoint. This requires a REST-type catalog, not the existing Glue-type catalog.

### Can't Manage Both in One Catalog

The first finding. A single Iceberg catalog cannot manage both S3 Table Bucket tables and existing S3 bucket Iceberg tables. LakeFormation permission policies are separated at the catalog level.

A separate Trino catalog is required.

```
Existing tables:     iceberg.raw_log.weblog_common
Table bucket tables: iceberg_managed.raw_log.weblog_common_managed
```

### DDL Limitations

Some DDL operations are not supported with REST-type catalogs.

- **CREATE TABLE**: Glue REST Catalog doesn't support the stage-create API that the Trino Iceberg connector depends on
- **RENAME TABLE**: Explicitly documented as unsupported in the Glue REST API

SELECT, INSERT, and other read/write operations work normally. Hive → Iceberg table redirection works, but Iceberg → Hive redirection is not supported with REST catalogs.

### LakeFormation Permissions

S3 Table Buckets have LakeFormation integration enabled by default. Every IAM role accessing the data — Trino workers, Kafka Connect, Spark — needs explicit LakeFormation permissions at the table, schema, or catalog level.

Without permissions, no error is thrown. The catalog simply appears to contain no tables. This makes debugging tricky.

---

## Trino Integration

Trino v471 added read support for S3 Table Buckets. Available in our current v476.

### Fault-Tolerant Mode Incompatibility

Writing to S3 Table Buckets with Trino's fault-tolerance (`retry_policy`) enabled causes errors. The internal path structure of table buckets differs from standard S3, breaking the file listing API. An issue is open in the Trino community.

Workaround: set `retry_policy` to `NONE` for the query session. Writes then work normally.

---

## Spark Integration

EMR Spark can query S3 Table Bucket Iceberg tables through the Glue REST Catalog endpoint.

Using AWS's open-source S3 Tables Catalog library requires EMR v7.5+, but using the Glue REST endpoint directly works on EMR v6.14.

Example Spark session configuration:

```properties
spark.sql.catalog.iceberg_managed=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.iceberg_managed.type=rest
spark.sql.catalog.iceberg_managed.uri=https://glue.ap-northeast-2.amazonaws.com/iceberg
spark.sql.catalog.iceberg_managed.rest.sigv4-enabled=true
spark.sql.catalog.iceberg_managed.rest.signing-name=glue
```

As with Trino, LakeFormation permissions must be granted to the EMR Spark IAM role. Missing permissions result in empty query results with no error.

---

## Kafka Connect Integration

We confirmed the Kafka Connect Iceberg sink connector can stream data to S3 Table Buckets through the Glue REST endpoint.

For CDC topics, Debezium source connectors include schema information in their messages, so automatic table creation and schema evolution work correctly. For schema-less data like web logs, tables must be pre-created since timestamp type inference isn't possible.

---

## Auto-Compaction Review

### Behavior

All tables created in S3 Table Buckets have fully managed compaction enabled by default. Default target file size is 512MB, adjustable between 64MB and 512MB.

The trigger conditions for auto-compaction are not documented. Job execution history can be checked via AWS CLI, but the trigger logic itself is opaque. Support cases are needed for specifics.

Compared to our hourly manual compaction batches, the trigger conditions appeared considerably more conservative.

### Cost

Beyond storage and read/write costs, auto-compaction incurs charges: $0.004 per 1,000 objects processed and $0.05 per GB processed.

For example, compacting 30,000 files at 5MB each (~146GB) costs approximately $7.44. Given the compute costs and maintenance overhead of manual compaction, this is acceptable.

### Not Needed for Append-Only Tables

Append-only log tables with time partitions receive data sequentially. Manual compaction handles them well. Given auto-compaction's conservative triggers, keeping the current approach is more cost-effective.

S3 Table Buckets make the most sense for CDC tables where updates and deletes happen across unpredictable partitions.

---

## CDC Sink Table PoC

Following the append-only log table PoC, we ran an additional PoC on a CDC table with continuous updates. The goal was to verify that auto-compaction doesn't conflict with CDC sink commits and improves maintainability.

The rollout plan: for new CDC table sink requests, run dual pipelines to both the existing S3 bucket and the S3 Table Bucket. Once stability is confirmed, remove the old pipeline.

---

## Takeaways

**Managed automation doesn't fit every workload.** Append-only tables are better served by existing manual compaction. Auto-compaction's value is for CDC tables with unpredictable partition-level updates.

**Catalog separation is unavoidable.** S3 Table Bucket catalogs and existing Glue catalogs can't be unified into a single Trino catalog. Users need to know the separate catalog name.

**Missing LakeFormation permissions show as empty results, not errors.** Hard to debug. A permissions checklist is essential.

**Don't forget past managed compaction failures.** Glue/LakeFormation managed compaction failed silently. There's no guarantee S3 Table Buckets won't do the same. Thorough PoC and dual-pipeline validation are the right approach.

**References:**
- [Amazon S3 Tables](https://aws.amazon.com/s3/features/tables/)
- [New Amazon S3 Tables: Storage optimized for analytics workloads](https://aws.amazon.com/blogs/aws/new-amazon-s3-tables-storage-optimized-for-analytics-workloads/)
- [How Amazon S3 Tables use compaction to improve query performance by up to 3x](https://aws.amazon.com/blogs/storage/how-amazon-s3-tables-use-compaction-to-improve-query-performance-by-up-to-3-times/)
- [Working with Amazon S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [Trino: Add read support for S3 Tables in Iceberg (v471)](https://github.com/trinodb/trino/pull/24815)
- [Trino: Fault tolerant mode fails with S3 Table Buckets](https://github.com/trinodb/trino/issues/25481)
- [Iceberg Connector - Trino Documentation](https://trino.io/docs/current/connector/iceberg.html)
- [AWS S3 Tables Catalog](https://github.com/awslabs/s3-tables-catalog)
