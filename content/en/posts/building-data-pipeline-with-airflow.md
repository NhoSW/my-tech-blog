---
title: "Building Reliable Data Pipelines with Apache Airflow: Lessons from Production"
date: 2026-02-23
draft: false
author: "Seungwoo Noh"
categories: [Data Engineering, Apache Airflow]
tags: [airflow, pipeline, orchestration, best-practices]
ShowToc: true
TocOpen: false
summary: "Practical lessons from building and maintaining production Airflow DAGs. Covers DAG design patterns, error handling strategies, and operational best practices."
cover:
  image: ""
---

## Introduction

Running Apache Airflow in a development environment is straightforward. Running it in production, where dozens of DAGs process terabytes of data on tight schedules and failures page you at 3 AM, is an entirely different challenge. Over the past few years I have built and maintained Airflow deployments that orchestrate everything from simple ETL jobs to multi-stage machine learning pipelines. This post distills the lessons that cost me the most sleep.

## DAG Design Best Practices

### Make Every Task Idempotent

The single most important rule is that every task must produce the same result regardless of how many times it runs. Airflow *will* retry your tasks. The scheduler *will* occasionally execute a task twice. If a task appends rows without checking whether they already exist, you end up with duplicates that silently corrupt downstream reports.

In practice this means using `INSERT ... ON CONFLICT` or `MERGE` statements instead of plain `INSERT`, partitioning output by execution date so that reruns overwrite the same partition, and never relying on wall-clock time inside a task when `{{ ds }}` or `{{ data_interval_start }}` is available.

### Keep Tasks Atomic and Focused

A task should do one thing. When a single operator extracts data from an API, transforms it, and loads it into a warehouse, any failure forces a full re-run of the entire sequence. Splitting this into three discrete tasks -- `extract`, `transform`, `load` -- means that a transient warehouse timeout only retries the load step.

### Model Dependencies Explicitly

Resist the temptation to chain tasks with `>>` in one long line. Group related tasks with `TaskGroup`, and use `ExternalTaskSensor` or the Dataset API to express cross-DAG dependencies. Implicit ordering through schedule alignment is fragile and breaks the moment execution times drift.

## Error Handling Strategies

**Retries with exponential backoff.** Most transient errors -- network timeouts, rate limits, brief service outages -- resolve themselves within minutes. Configure `retries=3` and `retry_delay=timedelta(minutes=5)` as a baseline, and increase the delay exponentially with `retry_exponential_backoff=True`.

**SLA monitoring.** Define `sla=timedelta(hours=2)` on critical tasks. When a task exceeds its SLA, Airflow fires an `sla_miss_callback` that can page your on-call engineer before downstream consumers even notice the delay.

**Alerting on failure.** Use `on_failure_callback` at the DAG level to send a Slack or PagerDuty notification the instant any task fails. Do not rely solely on the Airflow UI -- no one is watching it at 3 AM.

## A Concrete Example

Below is a simplified but production-style DAG that ingests order data from an API, stages it in cloud storage, and loads it into a data warehouse.

```python
from datetime import datetime, timedelta

from airflow import DAG
from airflow.decorators import task
from airflow.providers.amazon.aws.transfers.local_to_s3 import (
    LocalFilesystemToS3Operator,
)
from airflow.providers.postgres.operators.postgres import PostgresOperator

default_args = {
    "owner": "data-engineering",
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
    "retry_exponential_backoff": True,
    "on_failure_callback": notify_slack,  # defined elsewhere
    "sla": timedelta(hours=2),
}

with DAG(
    dag_id="orders_etl",
    default_args=default_args,
    schedule="@daily",
    start_date=datetime(2026, 1, 1),
    catchup=False,
    tags=["etl", "orders"],
) as dag:

    @task()
    def extract_orders(**context):
        """Pull orders for the logical date from the API."""
        logical_date = context["ds"]
        raw = fetch_orders_api(date=logical_date)  # idempotent fetch
        path = f"/tmp/orders_{logical_date}.parquet"
        raw.to_parquet(path)
        return path

    upload_to_s3 = LocalFilesystemToS3Operator(
        task_id="upload_to_s3",
        filename="{{ ti.xcom_pull(task_ids='extract_orders') }}",
        dest_key="raw/orders/{{ ds }}/orders.parquet",
        dest_bucket="data-lake-prod",
        aws_conn_id="aws_default",
        replace=True,  # overwrite ensures idempotency
    )

    load_to_warehouse = PostgresOperator(
        task_id="load_to_warehouse",
        postgres_conn_id="warehouse",
        sql="""
            COPY orders_staging FROM 's3://data-lake-prod/raw/orders/{{ ds }}/orders.parquet'
            IAM_ROLE 'arn:aws:iam::123456789012:role/redshift-copy'
            FORMAT AS PARQUET;

            -- Upsert into the final table for idempotency
            DELETE FROM orders WHERE order_date = '{{ ds }}';
            INSERT INTO orders SELECT * FROM orders_staging;
            TRUNCATE orders_staging;
        """,
    )

    extract_orders() >> upload_to_s3 >> load_to_warehouse
```

Key things to notice: the `replace=True` flag on the S3 upload ensures reruns overwrite the same object, the warehouse load deletes before inserting to avoid duplicates, and retry configuration lives in `default_args` so it applies uniformly.

## Operational Tips

**Test DAGs before deploying.** Run `python your_dag.py` to catch import errors, and use `dag.test()` (available since Airflow 2.5) to execute tasks locally against a real environment. Integrate these checks into CI so broken DAGs never reach production.

**Manage connections through environment variables or a secrets backend.** Storing credentials in the Airflow metadata database works for small teams, but it does not scale. HashiCorp Vault or AWS Secrets Manager as a backend keeps secrets centralized and auditable.

**Monitor resource consumption.** Watch scheduler loop duration, DAG parse time, and worker slot utilization. A DAG that takes 30 seconds to parse slows down the entire scheduler. Move heavy imports inside task callables and keep the top-level DAG file as lean as possible.

**Version your DAGs like application code.** Use Git, require code review, and tag releases. When a DAG produces incorrect data, you need to know exactly which version was running and be able to roll back quickly.

## Conclusion

Reliable Airflow pipelines are not built by accident. They emerge from deliberate design choices -- idempotent tasks, explicit dependencies, aggressive retry policies, and proactive monitoring. None of these practices are revolutionary on their own, but applying them consistently is what separates a pipeline that quietly does its job from one that wakes you up at night. Start with idempotency, add proper alerting, and iterate from there. Your future on-call self will thank you.
