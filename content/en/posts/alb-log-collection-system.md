---
title: "Building an ALB Log Pipeline with Filebeat, Flink, and Iceberg"
date: 2026-02-25
draft: false
categories: [Data Engineering]
tags: [alb, filebeat, flink, iceberg, sqs, kafka, kubernetes]
showTableOfContents: true
summary: "We needed to turn CSV ALB logs sitting in S3 into a queryable Iceberg table. The first attempt with Flink's filesystem connector ran out of memory. This post covers the redesign with Filebeat + SQS + Kafka, autoscaling, checkpoint tuning, and the operational surprises we hit along the way."
---

The cloud infra team came to us with a request. They wanted ALB logs available as a queryable table for post-incident analysis.

ALB logs were already landing in S3 as CSV files. When someone needed them, they'd query the files through Athena directly. Slow and painful. Text format meant query performance had a hard ceiling. Convert those logs to Parquet via Iceberg and you get something you can build a real-time dashboard on.

This post covers what we built and everything that went wrong along the way.

---

## First Design: Flink Filesystem Connector

We started with the simplest possible architecture.

```
S3 files → Flink (filesystem connector) → Iceberg table
```

Flink's filesystem connector watches an S3 bucket. New file appears, connector reads it, emits each line as a record. Attach an Iceberg sink and you're done.

The problem was **memory**. The filesystem connector keeps the entire list of processed files in Flink state. ALB logs generate tens of thousands of files per day. State kept growing. Memory kept climbing. Eventually it crashed. We hit a scalability wall and had to rethink the architecture.

---

## Current Architecture: S3 → SQS → Filebeat → Kafka → Flink → Iceberg

We pulled file monitoring out of Flink. A message queue and a lightweight collector sit in front now.

```
S3 bucket → SQS (file events) → Filebeat → Kafka → Flink → Iceberg
```

Each stage does one thing.

1. **S3 → SQS**: Bucket notification config pushes a message to SQS every time a new ALB log file lands
2. **SQS → Filebeat → Kafka**: Filebeat consumes SQS messages, fetches the S3 file directly, and sends each line to a Kafka topic
3. **Kafka → Flink → Iceberg**: Flink reads from Kafka, parses CSV, extracts metadata from the file path (account number, service name, role), and writes to an Iceberg table

Flink only talks to Kafka now. No more file lists in state.

---

## How We Ended Up on Filebeat

We didn't start with Filebeat. We tried Kafka Connect Filesystem Connector first. That failed. Here's what happened.

### Kafka Connect Filesystem Connector — Abandoned

This was a third-party connector with several problems.

- Not on Maven. Had to build from source
- Last GitHub commit was 4 years old. Java 8 codebase
- Messages were published more than once. Duplicate delivery with no clear fix
- S3 file move (cleanup) didn't work. `FileUtil.copy()` breaks on S3A filesystems

We decided that forcing an unmaintained tool to work was worse than picking something well-supported.

### Filebeat Bucket Listing Mode — Can't Scale

Filebeat supports two modes for S3 input. We tried **bucket listing mode** first. It periodically scans S3 for new files.

It worked. But it couldn't scale horizontally. Multiple Filebeat instances would process the same files because they don't know about each other. The official docs say it explicitly: bucket listing mode means a single instance with vertical scaling only.

There was another issue. The backup feature (`backup_to_bucket_arn`) that moves processed files to a separate bucket didn't work in most Filebeat versions with bucket listing mode. A GitHub Issue showed the fix had been merged then accidentally reverted. Only version 8.15.3 worked correctly.

### Switching to SQS Mode

SQS mode supports horizontal scaling. SQS handles message distribution, so multiple Filebeat pods process different files without overlap. S3 API costs go down too.

The tradeoff: no backfill of existing files. SQS mode only processes files that trigger events after setup. If you need historical files, you publish SQS events manually. Not a problem for our ALB log use case.

One more thing. SQS mode flat-out rejects `backup_to_bucket_arn`. Config validation fails at startup. Makes sense when you think about it — with SQS there's no need to separate new files from processed ones.

Final Filebeat config:

```yaml
filebeat.inputs:
  - type: aws-s3
    queue_url: https://sqs.ap-northeast-2.amazonaws.com/xxxx/s3-elb-logs-events
    file_selectors:
      - regex: ".*\\.log\\.gz"
    number_of_workers: 64
    decompress: true
    codec.line:
      format: message
output.kafka:
  hosts:
    - kafka-cluster:9092
  topic: cloudinfra.prod.streaming.alb-access-log.csv
  codec.format:
    string: '%{[aws.s3.object.key]} %{[message]}'
  compression: zstd
```

The key detail is `codec.format`. It prepends the S3 file path (`aws.s3.object.key`) to each message. Flink parses this path downstream to extract account numbers and service names.

---

## Autoscaling

### Filebeat: SQS-Based KEDA Scaling

We configured a KEDA ScaledObject using SQS Visible Messages as the trigger.

```yaml
spec:
  cooldownPeriod: 300
  maxReplicaCount: 128
  minReplicaCount: 1
  pollingInterval: 30
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueLength: '50'
        queueURL: https://sqs.ap-northeast-2.amazonaws.com/xxxx/s3-elb-logs-events
```

First attempt failed. KEDA's operator couldn't read SQS metrics due to IAM permissions. Setting `identityOwner: operator` in the ScaledObject makes KEDA use its own role instead of the pod's. We added `sqs:GetQueueAttributes` to that role and it worked.

### Flink: K8s Operator Autoscaling

Flink uses the Kubernetes Operator's built-in autoscaler.

```yaml
job.autoscaler.target.utilization: "0.75"
job.autoscaler.target.utilization.boundary: "0.15"
pipeline.max-parallelism: '480'
```

---

## Flink Checkpoint Tuning

We bumped the checkpoint interval from 1 minute to 5 minutes. A chain of failures followed.

### Heartbeat Timeout

Longer checkpoints meant TaskManagers couldn't send heartbeats in time. Default timeout was too short.

```yaml
heartbeat.interval: '60000'
heartbeat.timeout: '300000'
pekko.ask.timeout: 10m
```

### OOM: Java Heap Space

Longer checkpoint intervals meant more Kafka fetch data sitting in the heap. We bumped pod memory from 2G to 4G to 6G. Same crash every time.

The real problem: JVM Task Heap was only allocated 2.32GB even though the pod had plenty of memory overall. The Kafka fetch buffer filled that up. Fix was setting `taskmanager.memory.task.heap.size` explicitly.

```yaml
taskmanager.memory.task.heap.size: 4608m
taskmanager.memory.managed.size: 512m
taskmanager.memory.network.fraction: '0.02'
```

### Final Config

After several rounds of tuning we got the checkpoint interval up to 10 minutes.

```yaml
execution.checkpointing.interval: 10m
execution.checkpointing.timeout: 5m
heartbeat.interval: '60000'
heartbeat.timeout: '300000'
pekko.ask.timeout: 10m
taskmanager.memory.task.heap.size: 4608m
taskmanager.memory.managed.size: 512m
taskmanager.memory.network.fraction: '0.02'
```

---

## Compaction Failures and Dropping Upsert

Even after the 10-minute checkpoint interval, Iceberg compaction kept failing. We decided to eliminate every suspected cause.

1. **Created a new Iceberg table.** The old table's metadata might have been in a bad state. Fresh table, clean start.
2. **Removed upsert logic.** We'd been using UPSERT mode to prevent duplicates. That was causing compaction conflicts.

Dropping upsert means duplicates can appear. We checked. Duplicates did exist — but they weren't from the pipeline. The **source ALB logs themselves had duplicates**. We switched to append-only. Compaction started succeeding immediately.

---

## The Column Count Incident

One morning around 4 AM, Flink's CSV parsing started failing. ALB log columns had jumped from 31 to 34. AWS changed the format with no advance notice.

### First Fix: Dead Letter Queue

We applied a csv-dlq format to route parse failures to a DLQ topic. The app recovered, but a new problem appeared. Format errors flooded the TaskManager logs. **Ephemeral storage** filled up. Pods got evicted in a loop.

```
The node was low on resource: ephemeral-storage.
```

We fixed it by adjusting log4j to suppress format error logging.

```properties
logger.format.name = com.woowahan.dataservice.format
logger.format.level = ERROR
```

### Second Problem: DLQ Topic Overload

The DLQ topic received so many messages it put load on the entire Kafka cluster. We rolled back the DLQ approach and switched to `csv.ignore-parse-error` instead. Added 3 dummy columns to the source table definition so it accepts both 31-column and 34-column logs.

---

## Deduplication: UPSERT and Primary Keys

We initially used Iceberg UPSERT for deduplication. The challenge: ALB logs don't have a unique identifier.

We analyzed 467K log records and found that a combination of `file_path`, `time`, `http_request`, `client_addr`, `target_addr`, and `request_creation_time` was nearly unique. Only 1 duplicate existed across the entire dataset, and those 2 records were identical in every field — genuinely indistinguishable logs.

Setting a primary key in Iceberg required Flink SQL's `SET IDENTIFIER FIELDS`. Spark SQL doesn't support PRIMARY KEY syntax. Flink SQL doesn't support bucketing or hidden partitioning. Creating the table was harder than expected.

As described above, we ultimately dropped UPSERT for compaction stability.

---

## Monitoring

### Key Metrics

| Metric | What It Tells You |
|--------|-------------------|
| SQS Approximate Number Of Messages Visible | Files waiting to be processed |
| SQS Approximate Number Of Messages Not Visible | Files Filebeat is currently processing |
| Kafka consumer lag | How far behind Flink is |
| Flink job status | Whether the app is running |

### Alerts

- SQS: Visible Messages above threshold (backlog building up)
- Filebeat: High CPU/memory usage, pod not running
- Flink: Job transitions to non-running state

---

## Takeaways

Three lessons stand out.

**Don't use Flink for file watching.** The filesystem connector holds processed file lists in state. Run it long enough and memory blows up. Let SQS + Filebeat handle file discovery. Let Flink do what it's good at — stream processing.

**Upsert and compaction don't mix well.** This is the Iceberg position delete problem. For log data, append-only is far more stable operationally. Handle source-level duplicates on the consumer side.

**AWS can change log formats without warning.** ALB log columns went from 31 to 34 overnight. If your pipeline parses CSV, defensive options like `ignore-parse-error` aren't optional.

**References:**
- [Filebeat AWS S3 Input](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-aws-s3.html)
- [Flink Filesystem Connector](https://nightlies.apache.org/flink/flink-docs-release-1.19/docs/connectors/table/filesystem/)
- [Apache Iceberg - Flink Writes](https://iceberg.apache.org/docs/latest/flink-writes/)
- [KEDA AWS SQS Queue Scaler](https://keda.sh/docs/2.12/scalers/aws-sqs/)
