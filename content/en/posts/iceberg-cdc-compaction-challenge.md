---
title: "Iceberg - Why CDC Table Compaction Is So Tricky"
date: 2026-02-24
draft: false
categories: [Data Engineering]
tags: [iceberg, cdc, compaction, lakehouse, kafka-connect, deletion-vector, iceberg-v3]
showTableOfContents: true
summary: "Iceberg v2's row-level delete implementation, the difference between position and equality deletes, and why compaction commits keep failing when a real-time CDC sink is running. Covers how v3 Deletion Vectors change the picture and the workaround we settled on for v2 in production."
---

Run compaction on an Iceberg CDC table and you'll eventually see this:

```
org.apache.iceberg.exceptions.ValidationException:
Cannot commit, found new position delete for replaced data file
```

Append-only tables don't have this problem. CDC tables do. Understanding why requires digging into how Iceberg v2 handles deletes.

---

## Iceberg v2 Row-Level Deletes

### Two Ways to Delete Data: COW vs MOR

Iceberg offers two strategies for row-level changes.

**Copy-on-Write (COW)**: Rewrite the entire data file with the deleted rows removed. Great read performance, expensive writes. Good for batch updates on read-heavy tables.

**Merge-on-Read (MOR)**: Leave data files untouched. Record deletions in separate **delete files**. Fast writes, but reads pay the cost of merging originals with deletes at query time. This is what CDC/upsert tables use.

CDC pipelines have constant updates flowing in, so MOR is the natural fit. The catch: the delete files MOR produces are exactly what breaks compaction.

### Two Kinds of Delete Files

MOR uses two types of delete files.

**Equality delete**: Records the key values of rows to delete. "Delete any row with this PK." Can span multiple data files with a single delete file. Downside: readers must do key matching, which costs more at query time.

**Position delete**: Records a specific file path and row number. "Delete row 42 in A.parquet." Readers just skip that exact position — faster than key matching.

| Type | What It Records | Read Cost | Best For |
|------|----------------|-----------|----------|
| Equality delete | Key values | Higher (key matching) | PK-based bulk deletes |
| Position delete | File path + row number | Lower (position skip) | Frequent in-stream updates |

### Engine Support Varies

The Iceberg spec defines both types, but not every engine implements everything.

| Engine | Read Eq Delete | Read Pos Delete | Write Eq Delete | Write Pos Delete |
|--------|---------------|----------------|----------------|-----------------|
| Spark | Yes | Yes | No | Yes |
| Flink | Yes | Yes | Yes (UPSERT mode) | Situational |
| Trino | Yes | Yes | No | Yes |
| Athena | Yes | Yes | No | Yes |
| Hive/Impala | Yes | Yes | No | Yes |

Spark can read equality deletes but can't write them. Same for Trino and Athena — they only produce position deletes. Flink is the exception with equality delete writes in UPSERT mode.

This matters for compaction strategy.

---

## The Delta Writer: Where Delete Strategy Splits

The CDC sink uses iceberg-core's `BaseTaskWriter` to process records. The delete logic is interesting:

```java
public void deleteKey(T key) throws IOException {
    if (!internalPosDelete(asStructLikeKey(key))) {
        eqDeleteWriter.write(key);
    }
}
```

When building a single commit, the writer checks whether the record being deleted exists **in the current stream (commit)**:

- **Record is in this stream** → position delete
- **Record is not in this stream** → equality delete

Why not use equality delete for everything? Simpler logic, right? The reason is read performance. Position deletes are cheaper to apply during MOR reads. The write path optimizes for future read performance by preferring position deletes whenever possible.

This design choice is what causes compaction headaches.

---

## Snapshot Types and Commit Conflicts

### Four Snapshot Operations

Every Iceberg snapshot has an operation field describing what kind of change happened.

| Operation | Meaning | Typical Use |
|-----------|---------|-------------|
| `append` | Add data files | Batch loads, INSERT |
| `replace` | Swap data/delete files | **Compaction** |
| `overwrite` | Logical overwrite | **Upsert**, UPDATE, DELETE |
| `delete` | Remove data files or add delete files | DROP PARTITION, etc. |

Compaction is `replace`. It merges small files into larger ones and swaps them in. CDC sink is `overwrite`. It produces delete files as part of the upsert process.

### Optimistic Concurrency

Iceberg handles concurrent writes with optimistic concurrency. Think of it like Git.

1. Build a new metadata tree based on the current snapshot
2. Attempt an atomic commit (swap)
3. If another commit landed in between, run **validation**
4. Validation passes → commit succeeds. Fails → retry

The validation rules in step 3 determine which snapshot type combinations conflict and which auto-merge.

---

## Append-Only vs CDC: Why the Difference

### Append-Only Tables: Almost No Conflicts

Append-only tables just add data. No delete files. When compaction (`replace`) runs alongside new inserts (`append`), they touch different files. Auto-merge works fine. Like merging two Git branches that edited different files — no conflicts.

### CDC Tables: Position Deletes Are the Problem

CDC tables are different. Upserts produce delete files. Here's what goes wrong:

```
Timeline:
1. Compaction starts: reads file A, building replacement file B
2. CDC sink: deletes row 42 in file A (creates position delete)
3. Compaction finishes: tries to commit A → B replacement
4. Validation fails: "file A has a new position delete,
   replacing it would lose that delete"
```

```
ValidationException: Cannot commit, found new position delete
for replaced data file
```

Compaction already read file A and built a replacement. But a new position delete targeting file A appeared in between. Committing the replacement without that delete would break data correctness. Iceberg rejects the commit.

### Why Equality Deletes Cause Fewer Conflicts

Equality deletes say "delete rows with this key" — a logical declaration not bound to any specific file. When compaction replaces files, key-based deletes still apply to the new files.

Iceberg has a mechanism for this: compaction preserves the starting `sequence-number` ([PR #3480](https://github.com/apache/iceberg/pull/3480)), which lets equality deletes pass validation automatically. This is enabled by default.

Position deletes point to a specific file at a specific row. Replace that file and the position becomes meaningless. No automatic workaround possible.

---

## The Commit Interval Dilemma

"Can't we just tune the sink commit interval?" You can try. It doesn't solve the problem.

**Longer intervals**: More records per commit. Higher chance that the same record gets inserted then updated or deleted within one stream. The delta writer handles these as position deletes. More position deletes = more conflict potential.

**Shorter intervals**: Fewer position deletes per commit, but commits happen more often. If compaction can't finish before the next commit lands — and it only takes one position delete — conflict.

Neither direction gets you to zero conflicts.

---

## v2 Fix Attempts — All Failed

Several PRs tried to solve this within the v2 framework. **None were merged.**

| PR | Approach | Outcome |
|----|----------|---------|
| [#4703](https://github.com/apache/iceberg/pull/4703) | Skip position delete validation during compaction | Reviewers deemed it "dangerous," closed |
| [#4748](https://github.com/apache/iceberg/pull/4748) | Exploit same sequence number between Flink upsert pos-deletes and their data files | Closed |
| [#5760](https://github.com/apache/iceberg/pull/5760) | Add `min-data-sequence-number` to manifest entries for safer filtering | Closed by stale bot |
| [#7249](https://github.com/apache/iceberg/pull/7249) | `position-deletes-within-commit-only` snapshot property | Closed by stale bot |

No clean solution emerged within v2. The community converged on **fixing this structurally in v3.**

---

## Iceberg v3: How Deletion Vectors Change the Picture

The v3 spec was ratified in early 2025. Implementation started shipping with Iceberg 1.8.0 (February 2025). The key change: **Deletion Vectors (DVs).**

### Position Delete Files Are Gone

v3 bans new position delete file creation outright. DVs take their place.

> Position delete files must not be added to v3 tables, but existing position delete files are valid.

A DV is a **Roaring bitmap** stored in a Puffin file. Each bit marks whether a row in a specific data file has been deleted.

| | v2 Position Delete | v3 Deletion Vector |
|--|-------------------|-------------------|
| Format | Parquet (file path + row number columns) | Puffin (Roaring bitmap) |
| Per data file | Unbounded (N files can accumulate) | **At most 1** |
| On new delete | Append a separate delete file | Read existing DV, **merge, replace** |

### Why This Reduces Compaction Conflicts

The v2 conflict happens because position delete files exist independently of the data files they reference. Compaction replaces a data file, and any position delete file pointing at the old file becomes a dangling reference.

DVs work differently.

**Tightly coupled to data files.** A DV is a sidecar to its data file. When compaction rewrites a data file, the DV's deletions get absorbed into the new file. The old DV is removed alongside the old data file. No dangling references.

**No independent accumulation.** One data file gets at most one DV. New deletions merge into the existing DV rather than creating a separate file. The "new position delete for replaced data file" scenario from v2 doesn't arise the same way.

**Less compaction needed.** DVs are compact bitmaps, not Parquet files. They don't create the small-file problem that v2 position delete files did. Fewer compaction runs means a smaller window for conflicts.

### Row Lineage and Row-Level Conflict Detection

v3 also introduces **Row Lineage**. Every row gets a unique `_row_id` and `_last_updated_sequence_number`. Combined with DVs, this enables **row-level conflict detection** ([Issue #14613](https://github.com/apache/iceberg/issues/14613)).

In v2, touching the same data file meant a conflict. Period. In v3, two writers modifying different rows in the same file can auto-merge. If the CDC sink deletes row 42 while compaction reorganizes other rows, the commit can succeed without conflict.

### Not a Silver Bullet

OCC still applies. Two writers updating the same data file's DV at the same time will still conflict — one has to retry. But the retry is cheap: just re-merge a bitmap. No data re-scan needed. And once engines implement row-level concurrency using Row Lineage, even same-file conflicts can resolve automatically when different rows are affected.

### Engine Support (as of 2025)

| Engine | v3 DV Support |
|--------|--------------|
| Spark (Iceberg 1.8.0+) | Supported |
| AWS EMR / Athena / Glue | Announced November 2025 |
| Databricks | Supported (with row-level concurrency) |
| Trino (Starburst) | In progress |
| Flink | In progress |

Migrating is straightforward: `ALTER TABLE ... SET TBLPROPERTIES ('format-version' = '3')`. Existing position delete files stay valid. New deletes start using DVs.

---

## The v2 Workaround: Pause CDC During Compaction

If you're still on v2, the most reliable approach in production is straightforward: **stop the CDC sink while compaction runs.**

```
Compaction pipeline:

1. Pause CDC sink connector during low-traffic hours (e.g., early morning)
2. Run compaction (rewrite_data_files)
3. Resume CDC sink after compaction completes
```

Kakao's tech blog describes a similar approach — pausing real-time CDC ingestion at roughly 12-hour intervals for table maintenance.

### Things to Watch

- **Compaction can take longer than expected.** A full day's worth of CDC data might take hours to compact. Measure your window empirically.
- **Consider partition-level compaction.** Don't compact the entire table at once. Splitting by partition cuts wall time.
- **Clean up delete files separately.** Run `rewrite_position_delete_files` periodically for minor compaction. Watch for version-specific bugs — they've been reported.

---

## Operational Checklist

1. **Check your runtime version**: Verify `rewrite_data_files` validation rules and `rewrite_position_delete_files` support in your EMR/Spark/Flink/Trino version
2. **Monitor conflict rates**: Measure CDC commit frequency vs compaction duration. Track how often conflicts actually occur
3. **Build a metadata dashboard**: Visualize file counts, average sizes, and delete file accumulation from snapshot/manifest tables
4. **Secure a compaction window**: Set up nightly CDC pause or partition-level compaction schedules
5. **Evaluate v3 migration**: If your engine supports v3 DVs, migration eliminates most of the operational overhead described above

---

## Wrapping Up

Here's the short version:

- Iceberg v2's MOR path uses **position deletes to optimize read performance**
- Position deletes are bound to specific files, so they **conflict with compaction (replace)**
- Equality deletes get auto-resolved via the `sequence-number` mechanism. Position deletes don't
- Every attempt to fix this within v2 was closed without merging
- **v3 Deletion Vectors are the structural fix.** One bitmap per data file instead of unbounded position delete files. Row Lineage enables row-level conflict detection on top of that
- On v2, **the pragmatic answer is pausing CDC during a compaction window**

The v3 spec is ratified and engine support is expanding fast. If you're still on v2, planning the migration to v3 is the long-term play. Most of the operational overhead described in this post goes away once DVs are in place.

**References:**
- [Row-Level Changes on the Lakehouse: COW vs MOR in Apache Iceberg (Dremio)](https://www.dremio.com/blog/row-level-changes-on-the-lakehouse-copy-on-write-vs-merge-on-read-in-apache-iceberg/)
- [Apache Iceberg Spec - Row-level Deletes](https://iceberg.apache.org/spec/#row-level-deletes)
- [Apache Iceberg - Reliability (Concurrency)](https://iceberg.apache.org/docs/latest/reliability/)
- [PR #3480: Core: support rewrite data files with starting sequence number](https://github.com/apache/iceberg/pull/3480)
- [PR #4703: API: Optionally ignore position deletes in rewrite validation](https://github.com/apache/iceberg/pull/4703)
- [PR #7249: Avoid conflicts between rewrite datafiles and flink CDC writes](https://github.com/apache/iceberg/pull/7249)
- [Iceberg Table Loading and Operation Strategy by Log Type (Kakao Tech)](https://tech.kakao.com/posts/656)
- [Improve Position Deletes in V3 (Issue #11122)](https://github.com/apache/iceberg/issues/11122)
- [Row Lineage for V3 (Issue #11129)](https://github.com/apache/iceberg/issues/11129)
- [Row-level concurrency (Issue #14613)](https://github.com/apache/iceberg/issues/14613)
- [What's New in Apache Iceberg v3 (Google Open Source Blog)](https://opensource.googleblog.com/2025/08/whats-new-in-iceberg-v3.html)
- [Iceberg V3 Deletion Vectors on Amazon EMR (AWS Blog)](https://aws.amazon.com/blogs/big-data/unlock-the-power-of-apache-iceberg-v3-deletion-vectors-on-amazon-emr/)
