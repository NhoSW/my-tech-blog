---
title: "Building Iceberg Table Monitoring with Trino Meta-Tables and Prometheus Pushgateway"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [iceberg, trino, monitoring, grafana, prometheus, pushgateway, airflow]
showTableOfContents: true
summary: "As our Iceberg table count grew, we needed systematic monitoring for file health. We built dashboards using Trino meta-tables, Prometheus Pushgateway, and Grafana to track file counts, sizes, and partition distribution. Along the way we hit Trino bugs and performance issues worth documenting."
---

When you operate Iceberg tables, problems accumulate in places you can't see. Are small files piling up? Is compaction running properly? Is data skewing toward specific partitions? With a handful of tables you can check manually, but past a few dozen you need a monitoring system.

A [kakao tech blog post on Iceberg table operations strategy](https://tech.kakao.com/posts/694) prompted us to build this out properly.

---

## What to Monitor

Three things to track through Iceberg meta-tables:

- **Small file growth**: Small files accumulating degrades query performance. More files to scan means slower queries.
- **Optimization status**: Whether compaction is running correctly. File count and average file size trends tell the story.
- **Partition distribution**: Whether data is skewing toward specific partitions. Compare file counts and sizes per partition.

Periodically collecting the file count, average size, and partition distribution from the latest snapshot covers all of these.

---

## Architecture: Two Approaches

We evaluated two data source strategies for the monitoring system.

### Approach 1: Direct Trino Queries from Grafana

Connect a Trino datasource in Grafana. Every dashboard load submits meta-table queries directly to Trino.

Simple to implement, but the downsides were significant.

- Every dashboard view submits queries to Trino. Recomputing nearly identical results each time.
- Meta-queries on heavy tables take minutes. Dashboard loading becomes impractical.
- Trino's constraints make metric calculation logic hard to customize.
- The Grafana Trino plugin can't handle nested types like ROW.

### Approach 2: Prometheus Pushgateway

Similar to kakao's approach. Run meta-table queries in batch, push results to Pushgateway, and Prometheus pulls from there.

Clear advantages:

- No redundant query load on Trino.
- Full flexibility in metric calculation logic.
- Can use Spark or Glue Catalog API directly to work around Trino limitations.
- Naturally fits as a post-step to compaction jobs in Airflow.

We started with Approach 1 for speed, then progressively added Approach 2. Both now run in parallel for different use cases.

---

## Dashboard Design

### General Table Health Monitoring

Pushgateway-based dashboard. Tracks all Iceberg table health over time.

The `$files` meta-table only shows files referenced by the latest snapshot. It doesn't provide historical snapshots. But by periodically pushing meta-table query results to Pushgateway and pulling via Prometheus, each snapshot becomes a data point in a time series. You get trends over time.

With many tables, metric scales differ widely, making a single view hard to parse. We added regex text search to filter specific tables.

### Specific Table Deep Dive

Uses the Grafana Trino datasource directly. Select a table and it queries the `$partitions` meta-table to show file count and size per date partition at the current point in time.

The general dashboard tracks trends over time. This dashboard examines partition-level distribution at a specific moment.

---

## Trino Meta-Table Bugs and Cherry-Picks

We found two issues in our production Trino v451 and cherry-picked fixes from later versions.

### $files Not Showing Delete Files

Iceberg v2 delete files (positional and equality) weren't being recognized by the `$files` meta-table. Fixed in v455. We cherry-picked the commit into our fork's stage branch. Delete file stats appeared correctly after the fix.

### $files Missing Partition Field

Until v451, `$files` didn't include partition information for each file. This field is essential for per-partition file statistics. In v465, `partition`, `spec_id`, `sort_order_id`, and `readable_metrics` columns were added. Cherry-picked this as well.

---

## grafana-trino Plugin Limitations

The Grafana Trino plugin doesn't handle nested types or ROW-type fields. The `partition` field in the `$partitions` meta-table is a prime example. You have to specify `partition.log_ts_day` explicitly.

This makes it impossible to generically cover tables with additional partition columns beyond the time partition, or tables with non-day-level partitions. A fundamental limitation of the plugin.

---

## Heavy Table Meta-Query Performance

Meta-table queries on large tables like app logs and web logs were extremely slow. Too slow for practical use. We excluded these tables from monitoring targets for the time being.

These tables are large, but they also had `rewrite_manifests()` optimization not running properly. We expected that fixing manifest optimization would improve scan planning and thereby meta-query performance. However, even after normalizing `rewrite_manifests()`, meta-query performance showed no meaningful improvement.

### Spark vs Trino Meta-Table Query Performance

Spark's meta-table query performance is significantly better than Trino's, especially on heavy tables.

The reason: Spark uses iceberg-core's Table API to assemble DataFiles directly from Snapshot → ManifestList. Trino reads metadata files and JOINs them, resulting in higher scan overhead.

For batch pipelines pushing to Pushgateway, using Spark instead of Trino is a viable alternative. Spark also offers additional meta-tables like `all_files` for broader analysis.

---

## Metrics Pipeline

### Pushgateway Deployment

We deployed Prometheus Pushgateway to EKS via its Helm chart. Added a Pushgateway scraper to the Prometheus server configuration to collect metrics.

### Airflow Batch Pipeline

We built an Airflow DAG for the Iceberg table metrics push pipeline.

1. **Table extraction**: Automatically extract Iceberg-format tables via Glue API. Target schemas are managed as a list in the DAG code, excluding temp schemas.
2. **Dynamic task mapping**: For each extracted table, a task dynamically generated to run meta-queries and push metrics to Pushgateway.
3. **Metric definition**: Metrics like `iceberg_data_file_count` are labeled by `env`, `partition`, and `table`.

This can also be attached as a post-step to existing compaction jobs, adding metric push logic to the compaction task group.

### Listing Iceberg Tables Automatically

In our Trino version, filtering for Iceberg-format tables within a schema isn't possible. Both `SHOW TABLES` and `information_schema.tables` queries return all tables from the Glue Catalog regardless of format, even with table redirection disabled.

The `system.iceberg_tables` table added in v475 solves this by listing only Iceberg tables. Available after a version upgrade.

---

## DELETE File Observations

While building the monitoring system, we also examined delete file patterns across CDC tables.

Most CDC sink tables are effectively append-only because their source RDB tables are history-style tables. Record updates are rare, and subsequent compaction cleans them up, so delete files barely accumulate.

However, a few tables where records get updated or deleted shortly after insertion do generate positional delete files. These tables need separate consideration for compaction frequency and strategy.

---

## Future Trino Version Improvements

We're running v451. Here's what later versions bring for Iceberg monitoring.

| Version | Improvement |
|---------|-------------|
| v455 | Fixed `$files` meta-table not recognizing delete files |
| v465 | Added `partition` field to `$files` meta-table |
| v466 | Parallelized Glue Catalog queries for performance |
| v469 | Added `$all_entries` meta-table (corresponds to Spark's `all_entries`) |
| v470 | `$all_entries` bug fix, `optimize_manifests` procedure added |
| v475 | `system.iceberg_tables` table added (list Iceberg tables only) |

The v455 and v465 fixes were cherry-picked ahead of schedule. The rest will be picked up naturally with version upgrades.

---

## Takeaways

**Understand meta-table limitations.** `$files` only shows the latest snapshot. To track history, you need to periodically store results externally. Pushgateway is a natural fit.

**Trino's meta-queries are slow on heavy tables.** Spark is much faster due to its implementation approach. For batch pipelines, using Spark is reasonable.

**Combine direct queries and batch collection.** Overall table trends via Pushgateway batch. Specific table deep dives via direct queries. Splitting by use case was effective.

**If you run a Trino fork, cherry-picking is unavoidable.** When a needed bug fix only exists in a later version, you bring it in. Especially for monitoring, where inaccurate data is worse than no data.

**References:**
- [Iceberg Table Operations Strategy by Log Type - kakao tech](https://tech.kakao.com/posts/694)
- [Iceberg Connector - Trino Documentation](https://trino.io/docs/current/connector/iceberg.html)
- [$files delete file bug fix (v455)](https://github.com/trinodb/trino/pull/23142)
- [$files partition field (v465)](https://github.com/trinodb/trino/pull/24102)
- [Glue catalog query parallelization (v466)](https://github.com/trinodb/trino/pull/24110)
- [$all_entries metadata table (v469)](https://github.com/trinodb/trino/pull/24543)
- [optimize_manifests procedure (v470)](https://github.com/trinodb/trino/pull/24678)
- [system.iceberg_tables (v475)](https://github.com/trinodb/trino/pull/25136)
- [Prometheus Pushgateway](https://github.com/prometheus/pushgateway)
- [grafana-trino plugin](https://github.com/trinodb/grafana-trino)
