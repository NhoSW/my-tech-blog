---
title: "BigQuery Data Transfer + Airflow: Why We Create and Delete Transfers Every Batch"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [bigquery, airflow, data-transfer, gcp, aws, s3]
showTableOfContents: true
summary: "We built a pipeline to load S3 mart tables into BigQuery using Data Transfer Service. During PoC, DTS scheduling was managed by GCP. For production, we moved it into Airflow — creating a transfer object each batch tick and deleting it after completion. User feedback drove improvements: multi-day lookback windows, concurrent execution quota management via slot pools, and empty source path detection through GCP logging API."
---

We needed a pipeline to ingest Gold tables from S3 into BigQuery. The goal was making mart tables processed in the AWS data lake available for analysis in GCP BigQuery.

BigQuery Data Transfer Service (DTS) fits this purpose. It can load data directly from S3 to BigQuery with no additional compute cost. The question was where to manage DTS scheduling.

---

## PoC Approach and Its Limitations

During the PoC phase, the priority was getting things working quickly. We created DTS transfer objects as one-time Airflow jobs and let BigQuery itself handle the actual scheduling. Transfer schedules were viewed and managed in the BigQuery console.

It worked, but had operational problems. Transfer scheduling wasn't integrated into Airflow. Airflow couldn't tell whether a transfer had completed, couldn't manage dependencies with downstream tasks, and transfer failures were only visible in the BigQuery console.

An alternative was using BigQuery table sensors — detecting table updates to confirm transfers completed. But this is indirect. You can't distinguish between a failed transfer, one still in progress, or an empty source path.

We decided that creating and deleting transfer objects on every batch tick, entirely from Airflow, was the structurally correct approach.

---

## Design: Create Per Batch, Delete After Completion

The core idea is not keeping transfer objects permanently.

1. When an Airflow DAG batch tick starts, create a DTS transfer object
2. Wait for the transfer to complete (using async/deferrable operators)
3. After completion, delete the transfer object

The advantages are clear. Airflow manages the entire transfer lifecycle, so failure detection, retries, and downstream task dependencies all work through Airflow's existing mechanisms. No ghost transfer objects accumulate in the BigQuery console.

The default write mode is `WRITE_TRUNCATE`. Each time partition load overwrites existing data, guaranteeing idempotency. Running the same batch multiple times produces identical results.

### Implementation

Apache Airflow's Google Cloud Provider includes BigQuery DTS operators, but they didn't quite fit our use case and had unfixed bugs. We customized and fixed several issues, including them in our internal provider package (`airfilter-providers`). Bug fixes were also submitted as MRs to the upstream Airflow repository.

To use async operators, we added volume mounts to the Airflow triggerer component. The triggerer is the Airflow component that deferrable operators use to wait for external events — in this case, DTS transfer completion.

### User Interface

What users actually need to do is add an `S3ToBigQueryLoadConfig` object to the configuration file.

```python
S3_LOAD_CONFIG_BY_DAG_ID = S3ToBigQueryLoadConfigByDagIdDict(
    dataeng_bigquery_dts_load_dag_v1=[
        S3ToBigQueryLoadConfig(
            src_s3_path='s3://bucket/schema/table_name',
            daily_partition_field='part_dt',
            require_partition_filter=True,
            time_partition_expiration_days=365,
        ),
    ],
)
```

Specify the S3 source path, partition field, and expiration period — the framework handles the rest. Sink dataset and table names default to the last two path elements of the S3 path, with optional `sink_dataset` and `sink_table` overrides.

---

## DTS vs. Direct Spark Ingestion

Another option was using EMR Spark to create DataFrames and write directly to BigQuery. We chose DTS over this approach for two reasons.

**Cost:** Data loading through DTS incurs no compute charges. Network costs from AWS to GCP apply regardless of method, but DTS load compute is free. Running Spark jobs adds EMR cluster costs.

**Maintenance:** Spark ingestion requires managing Spark app creation, partition field removal logic, and BigQuery connector configuration. The DTS wrapper requires adding a single config object. Future plans include accepting configuration via config.yaml for even more simplification.

---

## User Feedback and Improvements

A team member migrated 23 tables and shared four pieces of feedback. All were valid operational concerns that only surface through real usage.

### Multi-Day Lookback

**Problem:** The initial implementation only supported loading the previous day's partition (D-1). But some tables update 14 days' worth of partitions daily. All 14 days needed refreshing, but only one day could be loaded per batch.

This pattern occurs with batch tables where the source system retroactively modifies historical data — the most recent N days' partitions get regenerated daily.

**Fix:** Added a `lookback_window_days` parameter to `S3ToBigQueryLoadConfig`. Default is 0 (previous day only). Setting it to 14 loads partitions from D-1 through D-14 on each batch tick.

### Concurrent Execution Quota

**Problem:** When 20+ table load tasks ran simultaneously, BigQuery's throughput quota caused some tasks to fail.

BigQuery has project-level limits on concurrent load jobs. Too many DTS transfers running at once exceeds the quota.

**Fix:** Rather than requesting a quota increase from BigQuery (which only pushes the problem out until more tables are added), we controlled concurrency from the Airflow side. Created a dedicated `bigquery_dts_pool` slot pool and mapped DTS transfer tasks to it. The pool size directly controls maximum concurrent executions.

### Empty Source Path Detection

**Problem:** When S3 source paths contained no files (due to user configuration errors), DTS reported success. BigQuery would have a table name created but no data. Users would see "success" while their table was actually empty.

This is DTS's inherent behavior. If the source path has no files, DTS considers its job done ("nothing to load") and returns success.

**Fix:** Configured `AirflowSkipException` to distinguish between "no files" and "actual failure." When files are missing, the task gets a "skipped" status — neither success nor failure.

One complication: the DTS TransferRun response object doesn't include information about whether source files existed. This detail is only recorded in GCP's logging service. Since Airflow's Google Cloud Provider doesn't include a GCP logging client hook, we implemented a custom hook to extract relevant logs from the GCP logging API.

### Timezone Handling

DTS scheduling timezone and Airflow's timezone can differ. If Airflow runs on KST but DTS creates transfers in UTC, date boundaries shift.

The initial implementation already included a timezone correction parameter, but user guidance was insufficient. We added UTC/KST documentation to the wiki.

---

## Partition Mapping

The partition structures of S3 Hive tables and BigQuery tables differ, requiring explicit mapping.

BigQuery tables support at most one partition field. But some S3 Hive tables use two-level partitioning — `LOG_DATE` and `LOG_HOUR`. For these tables, we mapped to BigQuery's pseudo column `_PARTITIONTIME`.

For example, data partitioned as `dt=2022-08-15/hour=3` maps to the `_PARTITIONTIME$2022081503` partition. Since `_PARTITIONTIME` is a datetime type, date and hour can be combined into a single field.

---

## WRITE_APPEND Mode Consideration

The default `WRITE_TRUNCATE` mode guarantees idempotency, but some tables need `WRITE_APPEND` — those that accumulate history.

With `WRITE_APPEND`, re-execution duplicates data. To guarantee idempotency, the transfer object must be retained rather than deleted after each batch. The same transfer object's same run won't duplicate loads. This was separated as a distinct design consideration.

---

## Takeaways

**Scheduling should be managed in one place.** When DTS self-scheduling and Airflow scheduling are separate, failure detection, retries, and dependency management all need dual implementations. Creating and deleting transfers per batch looks complex, but operationally it's the simplest approach.

**Don't blindly trust DTS "success."** DTS returns success even when the source has no files. Assuming "success = data was loaded" is dangerous. Separate verification that data actually exists is necessary.

**Quota limits are better managed client-side.** Requesting BigQuery quota increases is possible, but adding more tables later hits the limit again. Controlling concurrency via Airflow slot pools provides stable operation within quota boundaries.

**User feedback completes the design.** The initial implementation only considered "load one D-1 partition." Real users migrating 23 tables surfaced requirements — multi-day lookback, concurrent execution limits, empty path detection — that only emerge from production usage. Frameworks can't be completed without real-world feedback.

**References:**
- [BigQuery Data Transfer Service Pricing](https://cloud.google.com/bigquery/pricing#bqdts)
- [BigQuery: Introduction to Loading Data](https://cloud.google.com/bigquery/docs/loading-data)
- [BigQuery: Batch Loading Data](https://cloud.google.com/bigquery/docs/batch-loading-data)
- [BigQuery Quotas: Load Jobs](https://cloud.google.com/bigquery/quotas#load_jobs)
- [Airflow: Google Cloud BigQuery DTS Operators](https://airflow.apache.org/docs/apache-airflow-providers-google/stable/operators/cloud/bigquery_dts.html)
