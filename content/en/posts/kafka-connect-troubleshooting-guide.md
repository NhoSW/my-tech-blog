---
title: "Kafka Connect Troubleshooting: Common Issues and Solutions"
date: 2026-02-23
draft: false
author: "Seungwoo Noh"
categories: [Apache Kafka, Data Engineering]
tags: [kafka, kafka-connect, troubleshooting, debugging]
ShowToc: true
TocOpen: false
summary: "A practical guide to diagnosing and fixing common Kafka Connect issues. Covers connector failures, serialization errors, performance bottlenecks, and operational tips."
cover:
  image: ""
---

## Introduction

Kafka Connect is one of the most powerful components in the Apache Kafka ecosystem, providing a scalable and reliable way to stream data between Kafka and external systems. However, running Kafka Connect in production is far from a set-and-forget experience. Connector tasks silently fail, serialization mismatches corrupt pipelines, and rebalancing storms can bring an entire Connect cluster to its knees.

This post distills the issues I have encountered most frequently while operating Kafka Connect across dozens of production pipelines, along with the concrete steps I use to diagnose and resolve them.

## Common Connector Failures

### Tasks Stuck in FAILED State

The single most common issue is a connector whose tasks transition to `FAILED` without an obvious reason. Your first move should always be the Connect REST API:

```bash
# Check the status of a specific connector
curl -s http://localhost:8083/connectors/my-jdbc-sink/status | jq .

# Restart a single failed task (task 0)
curl -s -X POST http://localhost:8083/connectors/my-jdbc-sink/tasks/0/restart
```

The `status` response includes a `trace` field on failed tasks that contains the full Java stack trace. Read it carefully before restarting blindly -- a restart will not fix a misconfigured connection string or an expired credential.

### Serialization and Deserialization Errors

Serialization problems account for a disproportionate share of connector failures. The root cause is almost always a mismatch between the converter configured on the connector and the actual format of the data on the topic.

A typical mistake is pointing an Avro converter at a topic that contains JSON data, or vice versa. Make sure your connector configuration explicitly declares the correct converters:

```json
{
  "name": "orders-sink",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "topics": "orders",
    "connection.url": "jdbc:postgresql://db-host:5432/analytics",
    "connection.user": "connect_user",
    "connection.password": "${file:/opt/connect/secrets/db.properties:db.password}",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "io.confluent.connect.avro.AvroConverter",
    "value.converter.schema.registry.url": "http://schema-registry:8081",
    "insert.mode": "upsert",
    "pk.mode": "record_value",
    "pk.fields": "order_id",
    "auto.create": "false",
    "batch.size": 3000
  }
}
```

Notice that `key.converter` and `value.converter` are set at the connector level, overriding the worker defaults. This is a best practice -- it makes each connector self-documenting and immune to changes in the worker-level configuration.

### Schema Registry Problems

When using Avro or Protobuf converters, the Schema Registry becomes a critical dependency. If the registry is unreachable, every serialization call will fail. Common issues include:

- **Network partitions or DNS failures** between the Connect worker and the Schema Registry.
- **Subject-level compatibility violations** when a producer evolves a schema in a backward-incompatible way.
- **Authentication misconfiguration** when the registry sits behind basic auth or mTLS.

You can verify registry connectivity directly from the Connect host:

```bash
# List all registered subjects
curl -s http://schema-registry:8081/subjects | jq .

# Check the latest schema for a specific subject
curl -s http://schema-registry:8081/subjects/orders-value/versions/latest | jq .
```

## Performance Troubleshooting

### Slow Sink Consumers and Growing Lag

If your sink connector cannot keep up with the production rate, consumer lag will grow unboundedly. Start by checking the current lag:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server kafka:9092 \
  --describe \
  --group connect-orders-sink
```

Common remedies include:

1. **Increase `tasks.max`** to parallelize consumption across more threads. The upper bound is the number of partitions on the source topic.
2. **Tune `batch.size`** and `consumer.override.max.poll.records` to allow larger micro-batches, reducing per-record overhead.
3. **Check the downstream system** -- a slow database, an overloaded HTTP endpoint, or a saturated network link is often the real bottleneck, not Kafka Connect itself.

### Rebalancing Storms

In distributed mode, every time a connector or task is added, removed, or fails, the Connect cluster triggers a rebalance. Frequent rebalances (sometimes called "rebalancing storms") cause all tasks to pause, leading to spikes in consumer lag.

Mitigation strategies:

- Use **incremental cooperative rebalancing** by setting `connect.protocol=cooperative` in the worker configuration (available since Kafka 2.3).
- Increase `scheduled.rebalance.max.delay.ms` to give workers a grace period to rejoin before their tasks are redistributed.
- Avoid deploying connectors during peak traffic windows.

## Debugging Techniques

### Structured Log Analysis

Connect workers log extensively at the `INFO` level. For deeper investigation, temporarily raise the log level for a specific connector's logger:

```bash
# Set DEBUG logging for the JDBC sink connector
curl -s -X PUT \
  -H "Content-Type: application/json" \
  -d '{"level": "DEBUG"}' \
  http://localhost:8083/admin/loggers/io.confluent.connect.jdbc
```

This is a live change that does not require a worker restart, and it is invaluable for tracing exactly where a task fails in the put/flush cycle.

### JMX Metrics

Kafka Connect exposes JMX metrics under the `kafka.connect` MBean domain. The most useful metrics for troubleshooting include:

- `connector-total-task-count` and `connector-failed-task-count` -- instant visibility into task health.
- `sink-record-read-rate` and `sink-record-send-rate` -- throughput at the task level.
- `offset-commit-completion-rate` and `offset-commit-skip-rate` -- offset commit failures often precede data duplication or loss.

Export these to Prometheus via JMX Exporter, and build Grafana dashboards that alert on failed task counts and abnormal lag growth.

## Best Practices for Production Stability

1. **Pin converter settings per connector.** Never rely on worker-level converter defaults for production pipelines.
2. **Use externalized secrets.** The `${file:...}` config provider keeps credentials out of the Connect REST API responses and connector status endpoints.
3. **Deploy with infrastructure-as-code.** Store connector JSON configurations in version control and apply them via CI/CD. This ensures reproducibility and enables rollback.
4. **Implement a dead-letter queue.** For sink connectors, configure `errors.tolerance=all` and `errors.deadletterqueue.topic.name` to prevent a single poison message from halting the entire pipeline.
5. **Monitor offsets, not just lag.** A connector can appear healthy (no failed tasks) while silently stalling on offset commits. Track `offset-commit-completion-rate` in your alerting stack.
6. **Test schema changes in staging first.** Run your full Connect topology in a staging environment with realistic data volumes before promoting schema changes to production.

Kafka Connect is a reliable workhorse when configured and monitored correctly. Most production incidents trace back to a handful of recurring causes -- serialization mismatches, untuned batching, and missing observability. Addressing these systematically will save you from many late-night pages.
