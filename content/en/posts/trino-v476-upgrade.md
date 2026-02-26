---
title: "Trino v451 to v476: Upgrading a Production Cluster Across 25 Versions"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [trino, upgrade, kubernetes, blue-green, parquet, iceberg, java]
showTableOfContents: true
summary: "We upgraded Trino from v451 to v476 — 25 minor versions in one jump. Blue/Green deployment on our OLAP clusters caught two regressions: a Materialized View refresh bug and a Parquet read failure. We binary-searched the root cause down to a single v469 commit and got a community fix within a day."
---

We upgraded Trino from v451 to v476. Released June 5, 2025, v476 sits 25 minor versions ahead of what we were running. The change surface was massive, and we hit two regressions during rollout. One of them had never been reported in the community.

This post covers what we expected from the upgrade, what actually happened during deployment, and how we tracked down and resolved the regressions.

---

## Why v476

### Java 22 to Java 24

Trino aggressively adopts the latest JVM. It started requiring Java 22 at v447, Java 23 at v464, and Java 24 at v476. This isn't just a runtime version bump. Project Hummingbird is Trino's ongoing performance initiative that leverages Java's Vector API for vectorized execution. Vectorized Parquet decoding landed in v448, requiring 256-bit or larger vector registers — disabled on Graviton 2 instances (r6g, m6g) but automatically enabled on Graviton 3 and later.

Java 24's latest JIT compiler optimizations and memory management improvements directly impact query processing performance.

### S3 Table Bucket Read Support

Starting v471, the Iceberg connector can read from S3 Table Buckets. We were evaluating S3 Table Buckets as managed Iceberg tables for our CDC workloads, so Trino read support was a prerequisite. Integration goes through the Glue Data Catalog's Iceberg REST endpoint, requiring a separate REST-type catalog rather than the existing Glue-type catalog.

### Bucket Partition Push Down

Partitioning push down added in v468 optimizes scans on bucket-partitioned tables by pushing query conditions down to the partition level, reducing scan scope.

In our environment, app log and web log tables use bucket partitions on `screen_name`. These are large tables, so the performance improvement from bucket partition push down was expected to be significant.

### Iceberg Meta-Table Improvements

Fixes we had cherry-picked during our Iceberg monitoring buildout are now natively included in v476.

| Version | Improvement |
|---------|-------------|
| v455 | Fixed `$files` meta-table not recognizing delete files |
| v465 | Added `partition` field to `$files` meta-table |
| v466 | Parallelized Glue Catalog queries for performance |
| v469 | Added `$all_entries` meta-table (corresponds to Spark's `all_entries`) |
| v470 | `$all_entries` bug fix, `optimize_manifests` procedure added |
| v475 | `system.iceberg_tables` table added (list Iceberg tables only) |

Being able to use all these features without cherry-picking was reason enough for the upgrade.

### Alluxio File System Cache

Beyond the existing Alluxio-based local disk cache, a new feature allows integrating an external Alluxio cluster as a file system. This enables cross-catalog and cross-cluster cache sharing. Not something we planned to adopt immediately, but a useful option for future cache architecture improvements.

### Apache Ranger Plugin

The Apache Ranger authorization plugin was merged into the official Trino repository. Previously maintained in a separate repo, its inclusion in the main project simplifies version compatibility management.

---

## Deployment Strategy

### Blue/Green Gradual Rollout

Both our OLAP and BI cluster groups have Blue/Green pairs. The upgrade plan:

1. Deploy to one side (Green) of each cluster group
2. Monitor for one week
3. If no issues, apply to the other side (Blue)

If issues arise, we can immediately deactivate the upgraded cluster from Trino Gateway routing and roll back. This strategy is only possible because we operate Trino Gateway in front of our clusters. It proved essential — when we hit regressions, we rolled back with zero production impact.

We chose to start with OLAP-Green. OLAP has more diverse query patterns than BI, making it more likely to surface regressions quickly.

### Unresolved Issues from v451

Two issues from the v451 deployment remained open. We prepared mitigation plans in advance.

**Worker stuck issue**

During the v451 rollout, some workers would freeze under heavy load. The cause was `ThreadPerDriverTaskExecutor`, an experimental feature that had been added as default-enabled without documentation. The community issue was still open and unresolved.

We planned to disable `experimental.thread-per-driver-scheduler-enabled=false` again, same as before.

**Partition pruning regression**

Partition pruning failed when filter conditions contained type casts — for example, `WHERE date_column = CAST('2024-01-01' AS DATE)` wouldn't properly filter partitions.

For v451, we had reverted the causal commit directly. In the meantime, the community added an escape hatch setting that allows unsafe pushdowns. We planned to use this setting instead.

---

## First Deployment and Rollback

On June 23, 2025, after core business hours, we deployed v476 to OLAP-Green. Users were notified through the Zeppelin support channel.

### Materialized View REFRESH Stuck

Shortly after deployment, reports came in that Airflow DAGs refreshing Iceberg materialized views were getting stuck. The REFRESH queries were hanging in FINISHING state indefinitely.

Checking the Trino repo, we found that a v475 change titled "Cleanup previous snapshot files during materialized view refresh" had caused the regression. The timing was fortunate — a revert PR had been merged just the day before. We applied the revert commit to our fork and the issue was resolved.

### Parquet File Data Not Recognized

A more serious issue followed. A user reported incorrect query results from JupyterLab. Investigation revealed that a specific Hive table was returning no data at all.

The pattern: the user had written Parquet files to S3 first, then created a Hive table pointing to that location afterward. This worked perfectly on v451, but v476 couldn't see the pre-existing data files.

What we confirmed:

- The issue reproduced regardless of which engine created the data files (Trino v451, v476, or Spark)
- Glue API version (v1 vs v2) was irrelevant
- Running Spark's `MSCK REPAIR TABLE` or Trino's `system.sync_partition_metadata()` didn't help either

Given the severity, we **rolled OLAP-Green back to v451**. We would identify the root cause before redeploying.

---

## Binary Searching for the Root Cause

This issue had never been reported in the community. We had to find it ourselves.

Between v451 and v476, there were extensive changes to the metastore and Hive connector. Scanning release notes alone wouldn't identify the cause. We took a binary search approach.

We created a test notebook in Zeppelin to systematically check reproduction on each version.

```
v451 — OK
  ↓ (test midpoint)
v463 — OK
  ↓
v469 — Reproduces!
  ↓ (narrow range)
v466 — OK
v467 — OK
v468 — OK
v469 — Reproduces!
```

**The regression started at v469.**

Next was identifying which commit within v469 caused it. We reviewed v469's release notes for Hive connector changes, then reverted each related PR one at a time, testing reproduction after each revert.

---

## Root Cause: Parquet Footer Optimization Meets Legacy pyarrow

The culprit was a Parquet footer parsing optimization PR merged in v469. This PR trusted the `RowGroup.getFile_offset` value in Parquet file footer metadata and used it to optimize read ranges.

The problem: not all Parquet files have accurate row group offsets. The affected files were written by pyarrow v5.0.0 (parquet-cpp-arrow v5.0.0, via pandas). This legacy version recorded inaccurate row group offsets.

Additional testing confirmed:

- Files written by current pyarrow versions were not affected
- The offset error only manifested when row groups exceeded a certain size. Small files didn't reproduce
- Pre-v469 Trino didn't use `getFile_offset`, so inaccurate offsets were harmless

---

## Community Report and One-Day Fix

We organized our findings — the causal PR, reproduction steps, and the specific file characteristics (legacy pyarrow version, row group size threshold) — and shared them in the Trino community Slack. We also filed an issue titled "Incorrect results on parquet files written by parquet-cpp-arrow version 5.0.0."

The original optimization PR author identified the root cause immediately. The optimization assumed `RowGroup.getFile_offset` was always accurate, but legacy writers could produce incorrect values.

The fix: use the first page offset from `ColumnChunk` instead of `RowGroup.getFile_offset`, adding defensive logic for files with inaccurate offset metadata. The fix PR appeared within a day of our report and was merged for v477.

We cherry-picked the fix into our fork, deployed to staging to verify, and applied it to production.

---

## Redeployment and Results

With both issues resolved, we resumed the rollout.

| Date | Cluster | Action |
|------|---------|--------|
| June 23 | OLAP-Green | Initial v476 deploy → issues found, rolled back |
| June 26 AM | OLAP-Green | Redeployed with fixes |
| June 26 PM | OLAP-Blue | v476 deployed |

The BI cluster group was split into a separate ticket — deploying on Friday would mean weekend incident response risk. BI deployment proceeded the following Monday.

Post-deployment monitoring showed no issues with cluster health or query failure metrics. No additional user reports came in.

While the observation period was still short, the v476-deployed cluster showed relatively higher query throughput. With both clusters at nearly identical worker counts and the gateway routing based on running query count, this means v476 was handling more queries with lower load. It's possible that heavy queries happened to land on the other cluster, but the data suggests genuine performance improvement in the new version.

---

## Takeaways

**Without Blue/Green, this would have been a production incident.** Deploying to one cluster first meant we could discover regressions and immediately roll back while routing traffic to the other side. For a 25-version jump, Blue/Green isn't optional — it's essential.

**Binary search is the go-to for regression debugging.** Reviewing changes across 25 versions individually isn't feasible. Testing reproduction at midpoints narrows the range by half each time. Five or six tests are enough to pinpoint the version. From there, reverting individual commits within that version identifies the exact PR.

**A well-structured issue report gets fast results.** We shared the causal PR, reproduction steps, and specific conditions (legacy pyarrow version, row group size threshold). The original PR author identified the root cause and submitted a fix within a day. Filing "it doesn't work" versus sharing root cause analysis makes an enormous difference in response time.

**Watch for experimental features enabled by default.** `ThreadPerDriverTaskExecutor` caused issues in v451 and was still default-enabled in v476. The community issue remains open. Always check release notes for experimental feature changes during upgrades.

**Turn upgrade lessons into a checklist.** We had prepared mitigations for two v451 issues (worker stuck, partition pruning regression) in advance, which made this upgrade smoother. Recording issues and mitigations from each upgrade pays off for the next one.

**References:**
- [Trino v475 Release Notes](https://trino.io/docs/current/release/release-475.html)
- [Trino v476 Release Notes](https://trino.io/docs/current/release/release-476.html)
- [Require Java 24 (v476)](https://github.com/trinodb/trino/pull/25815)
- [Add partitioning push down (v468)](https://github.com/trinodb/trino/pull/23432)
- [Add Apache Ranger authorizer plugin](https://github.com/trinodb/trino/pull/22675)
- [Alluxio file system support](https://trino.io/docs/current/object-storage/alluxio.html)
- [Add read support for S3 Tables in Iceberg (v471)](https://github.com/trinodb/trino/pull/24815)
- [ThreadPerDriverTaskExecutor issue](https://github.com/trinodb/trino/issues/21512)
- [Partition pruning regression](https://github.com/trinodb/trino/issues/22268)
- [Escape hatch for unsafe pushdowns](https://github.com/trinodb/trino/pull/22987)
- [MV refresh revert PR](https://github.com/trinodb/trino/pull/26051)
- [Parquet footer optimization (v469, regression cause)](https://github.com/trinodb/trino/pull/24618)
- [Parquet file_offset issue report](https://github.com/trinodb/trino/issues/26058)
- [Parquet fix PR (v477)](https://github.com/trinodb/trino/pull/26064)
