---
title: "S3 Table Buckets PoC: Evaluating Managed Iceberg for CDC Workloads"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [iceberg, s3, aws, trino, spark, kafka-connect, compaction, managed-service]
showTableOfContents: true
summary: "AWS S3 Table Buckets offer managed Iceberg tables with automatic compaction. We ran a PoC to see if they could solve our CDC table compaction problem. We validated Trino, Spark, and Kafka Connect integration, examined auto-compaction behavior, and assessed costs. The conclusion: not a fit for every table, but valuable specifically for CDC workloads with unpredictable partition-level updates."
---

S3 Table Buckets launched in the Seoul region in March 2025. In one line: managed Iceberg tables. AWS handles automatic compaction and snapshot management, and claims higher TPS and throughput than standard S3 buckets.

The reason this caught our attention was clear. CDC sink table compaction had been a persistent headache.

Here's the compaction problem with CDC tables. Even with time partitions, non-history source tables generate updates and deletes across all partitions. For example, if a user who signed up three years ago changes their nickname today, that generates a delete marker and a new record in the three-year-old partition. Setting the compaction window to the last 5 days leaves files outside that range uncompacted. Extending it to 30 days doesn't solve it either. Full-range compaction can cost tens to hundreds of dollars depending on table size, and the execution time is long enough to impact other queries.

Could automatic compaction solve this? We ran a PoC.

---

## Managed Service Concerns

Alongside the excitement, there were genuine concerns. Three things worried us.

First, the black box problem. When issues arise in a managed service, you can't look inside. When compaction doesn't run, your options for understanding why are limited. On top of that, the service had only been out for a few months, so real-world usage reports and troubleshooting references were scarce.

Second, a painful past experience. S3 Table Buckets weren't actually AWS's first attempt at managed Iceberg compaction. Glue/LakeFormation had previously offered a managed compaction feature, which internally ran as Glue-based Spark procedure invocation jobs. The problem: on heavy tables, it failed silently. No error logs, just no compaction. Hundreds of files would pile up, but the Glue job history showed "success." We eventually went back to running our own compaction batches. The same thing could happen again.

Third, cost. S3 Table Bucket storage is more expensive than standard S3 buckets, with additional charges for automatic compaction. We needed to verify whether the convenience of managed compaction justified the premium.

---

## Glue REST Catalog Integration

S3 Table Buckets are accessed through the Glue Data Catalog's Iceberg REST endpoint. This requires a REST-type catalog (`type=rest`), separate from the existing Glue-type catalog (`type=glue`).

This architectural decision cascaded into several downstream constraints.

### Can't Manage Both in One Catalog

The first finding from the PoC. A single Iceberg catalog cannot manage both S3 Table Bucket tables and existing S3 bucket Iceberg tables. The root cause is that LakeFormation permission policies are separated at the catalog level. REST-type catalogs and Glue-type catalogs are governed by different permission hierarchies in LakeFormation.

This directly affects user experience. A separate Trino catalog must be added, and users need to know which catalog holds which tables.

```
Existing tables:     iceberg.raw_log.weblog_common
Table bucket tables: iceberg_managed.raw_log.weblog_common_managed
```

Joining existing tables with table bucket tables requires cross-catalog queries. Not a common pattern, but during migration validation we frequently needed to compare data between the two, which was cumbersome.

### DDL Limitations

Some DDL operations are not supported with REST-type catalogs. This is a Glue REST API limitation, not a Trino constraint.

- **CREATE TABLE**: The Glue REST Catalog doesn't support the stage-create API that the Trino Iceberg connector depends on. Tables must be created through the AWS console or Spark.
- **RENAME TABLE**: Explicitly documented as unsupported in the Glue REST API.

SELECT, INSERT, and other read/write operations work normally. Hive-to-Iceberg table redirection works, but Iceberg-to-Hive redirection is not supported with REST catalogs.

The CREATE TABLE limitation is particularly inconvenient operationally. Each new CDC table requires opening a Spark session to create it. In practice though, the Kafka Connect Iceberg sink connector handles table auto-creation, so we only need to pre-create the schema and table bucket namespace.

### LakeFormation Permissions

S3 Table Buckets have LakeFormation integration enabled by default. Every IAM role accessing the data — Trino workers, Kafka Connect, Spark — needs explicit LakeFormation permissions at the table, schema, or catalog level.

The trickiest part is the failure mode when permissions are missing. No error is thrown. The catalog simply appears empty. `SHOW TABLES` returns an empty result, and `SELECT` returns "Table not found." There's no way to tell whether the table genuinely doesn't exist or whether you lack permissions.

We lost considerable time on this during early PoC work. We reconfigured the Trino catalog three times before realizing it was a missing LakeFormation grant. After that experience, we created a checklist that starts with LakeFormation permissions for any S3 Table Bucket issue.

---

## Trino Integration

Trino v471 added read support for S3 Table Buckets. Both reads and writes are available in our current v476.

### Fault-Tolerant Mode Incompatibility

An important constraint discovered during the PoC. Writing to S3 Table Buckets with Trino's fault-tolerance (`retry_policy`) enabled causes errors.

The suspected cause is the internal path structure of table buckets. Table buckets use a different path structure than standard S3. Trino's fault-tolerant mode lists intermediate write results to determine retry behavior, and this listing API is incompatible with the table bucket path structure. An issue is open in the Trino community but remains unresolved.

The workaround is setting `retry_policy` to `NONE` for the query session. Disabling fault-tolerant mode allows writes to succeed. However, since this is a session-level setting, users must remember to set it when writing to table buckets. Batch jobs can have it hardcoded, but interactive queries require manual configuration each time.

---

## Spark Integration

EMR Spark can query S3 Table Bucket Iceberg tables through the Glue REST Catalog endpoint.

Two integration paths exist. AWS's open-source S3 Tables Catalog library requires EMR v7.5+, but using the Glue REST endpoint directly works on EMR v6.14. Since our environment still runs EMR v6.14, we used the direct REST endpoint approach. It works without issues.

Spark session configuration:

```properties
spark.sql.catalog.iceberg_managed=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.iceberg_managed.type=rest
spark.sql.catalog.iceberg_managed.uri=https://glue.ap-northeast-2.amazonaws.com/iceberg
spark.sql.catalog.iceberg_managed.rest.sigv4-enabled=true
spark.sql.catalog.iceberg_managed.rest.signing-name=glue
```

As with Trino, LakeFormation permissions must be granted to the EMR Spark IAM role. The failure mode is identical: no error, just empty results. If `SHOW TABLES` returns nothing in Spark, it's almost certainly a LakeFormation permissions issue.

---

## Kafka Connect Integration

This was the most critical integration to validate. Our primary CDC sink pipeline runs through Kafka Connect.

We confirmed the Kafka Connect Iceberg sink connector can stream data to S3 Table Buckets through the Glue REST endpoint.

For CDC topics, Debezium source connectors include schema information in their messages, so automatic table creation and schema evolution work correctly. We verified that when columns are added to the source table, they're automatically added to the sink-side Iceberg table as well.

However, for schema-less data like web logs, timestamp type inference isn't possible, so tables must be pre-created. If a timestamp field arrives as a plain string, the Iceberg connector creates it as a `string` type. Changing it to `timestamp` later requires recreating the table. This isn't an S3 Table Bucket-specific issue — it's an Iceberg sink connector limitation — but since CREATE TABLE doesn't work from Trino on table buckets, table recreation becomes more cumbersome.

---

## Auto-Compaction Review

### Behavior

All tables created in S3 Table Buckets have fully managed compaction enabled by default. The default target file size is 512MB, adjustable between 64MB and 512MB.

The trigger conditions for auto-compaction were what we most wanted to understand from the PoC. The bottom line: trigger conditions are not documented. Job execution history can be checked via AWS CLI, but the actual trigger logic — "under what conditions does compaction start?" — is not publicly available. Specifics require opening a support case.

From our observations, trigger conditions appeared considerably more conservative than our hourly manual compaction batches. Manual compaction runs as soon as files exceed a threshold count; auto-compaction waited until files accumulated significantly more. The frequency itself was lower.

This is a trade-off depending on perspective. Conservative triggers reduce unnecessary compaction costs, but the small files that accumulate in the meantime can impact query performance. For CDC tables with continuously accumulating small files, more aggressive compaction might be preferable.

### Cost

Beyond storage and read/write costs, auto-compaction incurs two types of charges:

- $0.004 per 1,000 objects processed
- $0.05 per GB processed

A concrete example: compacting 30,000 files at 5MB each (~146GB):
- Object processing: 30,000 / 1,000 × $0.004 = $0.12
- Data processing: 146 × $0.05 = $7.30
- Total: approximately $7.44

For manual compaction using EMR Spark jobs or Trino queries, processing the same volume can cost more in EC2 instance hours alone. Factor in the operational overhead — batch job management, failure monitoring, retry logic — and auto-compaction's cost is quite acceptable.

### Not Needed for Append-Only Tables

One conclusion became clear through the PoC: not every table needs S3 Table Buckets.

Append-only log tables receive data sequentially by time partition. New files only land in recent partitions; historical partitions are never touched. For these tables, existing manual compaction works perfectly. Hourly batches targeting only recent partitions have clear scope and predictable cost.

Given auto-compaction's conservative triggers, manual compaction is actually more cost-effective and performant for append-only tables.

Where S3 Table Buckets truly add value is CDC tables — workloads where updates and deletes happen across unpredictable partitions, making it impossible to define a fixed compaction scope. For these tables, the convenience of automatic compaction justifies the additional cost.

---

## CDC Sink Table PoC

Following the append-only log table PoC, we ran an additional PoC on a CDC table with continuous updates. Two questions needed answers.

First, does auto-compaction work without conflicting with CDC sink commits? If Kafka Connect is committing data while auto-compaction runs simultaneously, conflicts could arise. Iceberg's optimistic concurrency control handles this in theory, but we needed to verify it in practice.

Second, does maintainability actually improve compared to manual compaction? Even if costs are similar, eliminating the need for operators to manage compaction batches is valuable in itself.

The production rollout plan: when new CDC table sink requests come in, run dual pipelines to both the existing S3 bucket and the S3 Table Bucket. Sink the same Kafka topic to both destinations simultaneously, validating data consistency and auto-compaction behavior. Once stability is confirmed over a sufficient period, remove the old S3 bucket pipeline.

Running dual pipelines doubles the cost — but after our experience with Glue/LakeFormation's silent compaction failures, we consider this verification cost an insurance premium.

---

## Takeaways

**Managed automation doesn't fit every workload.** Append-only tables are actually better served by existing manual compaction. Auto-compaction's conservative triggers and additional costs make manual compaction more rational for workloads with clear, predictable scope. Auto-compaction adds value specifically for CDC tables with unpredictable partition-level updates and deletes.

**Catalog separation is unavoidable.** S3 Table Bucket REST catalogs and existing Glue catalogs can't be unified into a single Trino catalog. LakeFormation permission hierarchies are separated at the catalog level. Users need to know the separate catalog name, and cross-catalog queries are required for joins between the two.

**Missing LakeFormation permissions show as empty results, not errors.** This was the single biggest time sink during the PoC. Instead of throwing "permission denied," it shows "no tables found." When any new IAM role needs access to S3 Table Buckets, checking LakeFormation permissions must be the first item on the checklist.

**Don't forget past managed compaction failures.** Glue/LakeFormation's managed compaction failed silently. There's no guarantee S3 Table Buckets won't do the same. Thorough PoC and dual-pipeline validation are necessary for that reason. Trust comes from observed stability, not from service marketing pages.

**References:**
- [Amazon S3 Tables](https://aws.amazon.com/s3/features/tables/)
- [New Amazon S3 Tables: Storage optimized for analytics workloads](https://aws.amazon.com/blogs/aws/new-amazon-s3-tables-storage-optimized-for-analytics-workloads/)
- [How Amazon S3 Tables use compaction to improve query performance by up to 3x](https://aws.amazon.com/blogs/storage/how-amazon-s3-tables-use-compaction-to-improve-query-performance-by-up-to-3-times/)
- [Working with Amazon S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [Trino: Add read support for S3 Tables in Iceberg (v471)](https://github.com/trinodb/trino/pull/24815)
- [Trino: Fault tolerant mode fails with S3 Table Buckets](https://github.com/trinodb/trino/issues/25481)
- [Iceberg Connector - Trino Documentation](https://trino.io/docs/current/connector/iceberg.html)
- [AWS S3 Tables Catalog](https://github.com/awslabs/s3-tables-catalog)
