---
title: "Why BigQuery Hangs in Superset: The gevent and gRPC Incompatibility"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [superset, bigquery, redash, gunicorn, gevent, grpc, troubleshooting]
showTableOfContents: true
summary: "We integrated BigQuery with both Redash and Superset for per-user BI access. Redash had a known schema browsing limitation. Superset hit a much worse problem — queries executed successfully on BigQuery but results never came back, blocking indefinitely. The root cause: gunicorn's gevent worker monkey-patches Python's threading primitives, which deadlocks gRPC calls inside the BigQuery SDK. Switching the worker type to gthread resolved it. We contributed a docs fix upstream."
---

A request came in to enable BigQuery access through our BI tools based on per-user permissions. We proceeded to integrate BigQuery with both Redash and Superset, the two BI platforms we operate.

Redash was relatively straightforward. It had a known limitation where schema browsing doesn't work, but queries ran fine. Superset was a different story. Queries were successfully submitted to BigQuery but results never came back — infinite blocking. Tracing the cause led us deep into Python's concurrency model where a fundamental conflict was happening.

---

## Redash BigQuery Integration

Integrating BigQuery with Redash is simple. Add BigQuery as a Data Source and register the service account JSON key.

Query execution works normally. But there's one problem: the web UI can't browse dataset and table schemas. The sidebar shows no table list.

This is a known issue in the Redash community. BigQuery's metadata API response structure doesn't match Redash's schema parsing logic. It's been reported for years but never fixed. With Redash's project activity declining, a fix is unlikely.

Schema browsing has to be done directly in the BigQuery web console. It's inconvenient for query writing, but since we planned to use Superset as the primary BigQuery BI tool, we left Redash in this state.

---

## Superset BigQuery Integration

### Dependency Installation

Integrating BigQuery with Superset requires adding `sqlalchemy-bigquery` and `pybigquery` dependencies. Since we run Superset as a custom Docker image, we added the dependencies at the image build stage.

The BigQuery packages caused version conflicts with existing Superset dependencies. Aligning versions of `google-cloud-bigquery`, `google-auth`, `grpcio`, and others required some work. After resolving the dependency conflicts, we built and deployed the image.

Unlike Redash, Superset correctly displayed BigQuery dataset and table schemas in the web UI. This alone was sufficient reason to use Superset as the BigQuery BI tool.

### Queries Start Blocking

After finishing dependency installation and connection configuration, we started testing. Something was wrong.

Queries executed in SqlLab returned results normally. But running queries in the chart development screen produced no response. After about 60 seconds, a Gateway Timeout occurred.

Checking the GCP BigQuery console, even the blocked queries had been successfully submitted and completed on the BigQuery side. The queries ran — but results weren't making it back to Superset.

At this point we searched Superset's GitHub Issues. An issue reporting identical symptoms existed. But the workarounds suggested in the thread didn't resolve our case.

---

## Why SqlLab Works but Charts Don't

We began debugging directly, tracking what differs between SqlLab and the chart development screen.

### Async Mode Behavior Differences

With Superset's async mode (`ASYNC_QUERIES_REDIS`) enabled, SqlLab queries are delegated to Celery workers. Celery workers execute queries in separate processes and store results in Redis. The web server just polls for results and returns them to the user.

But in the dashboard and chart development environment, even with async mode enabled, query submission logic runs directly in the web server process (gunicorn). We confirmed this difference by examining pod logs.

Enabling the `GLOBAL_ASYNC_QUERIES` feature flag causes some additional UI paths to use Celery workers, but operations like force refresh (cache bypass) still submit queries from the web server.

Summary:

| Execution Context | Query Execution Location | Result |
|------------------|------------------------|--------|
| SqlLab | Celery worker | Works |
| Chart development | gunicorn web server | Blocks |
| Force refresh | gunicorn web server | Blocks |

**Works when running on Celery workers, blocks when running on the gunicorn web server.** The problem is in gunicorn.

### Code-Level Blocking Point

We traced through the code to identify the exact blocking point. It was in Superset's `db_engine_specs/bigquery.py`, where the `python-bigquery` SDK method is called. On Celery workers, this call returns normally. On the gunicorn web server, it hangs indefinitely.

We were now certain there was a compatibility issue between the `python-bigquery` library and the gunicorn server.

---

## Root Cause: gevent's Monkey Patching Deadlocks gRPC

### gunicorn Worker Types

gunicorn supports several worker types. We were using `gevent`.

- **sync**: One request per worker. Simple but low concurrency.
- **gthread**: Thread pool per worker. Uses Python's native threading.
- **gevent**: Greenlet-based coroutines. Can handle thousands of concurrent connections with minimal resources.

gevent achieves high concurrency through a special technique. When the Python process starts, it dynamically replaces standard library I/O, socket, and threading modules with its own implementations. This is called **monkey patching**. When you `import socket`, you actually get gevent's socket.

### How Monkey Patching Deadlocks gRPC

The BigQuery Python SDK (`google-cloud-bigquery`) uses gRPC internally. gRPC has a C core with Python bindings on top, and it relies on native OS threads.

The problem occurs when gevent monkey-patches Python's `concurrent.futures.ThreadPoolExecutor`. The BigQuery SDK uses `ThreadPoolExecutor` while waiting for gRPC call results. gevent's patched version of this executor is incompatible with gRPC's native threads. The result: gRPC calls complete at the C level, but the Python side never receives the result and waits forever.

This wasn't a new problem. The `python-bigquery` repo had a past issue where the `to_dataframe()` method blocked under gevent. That specific method was fixed, but the same class of problem manifested in other gRPC call paths within the BigQuery SDK.

Related issues exist in both gevent's and gRPC's GitHub repos. The gevent + gRPC combination is structurally difficult to make compatible.

### Why Celery Workers Are Fine

Celery workers run as separate processes from gunicorn. In the default configuration, Celery workers use `prefork` mode, where gevent's monkey patching is not applied. Python's standard threading works as-is, so there's no gRPC compatibility issue.

Monkey patching only applies in the gunicorn web server process — which is exactly why only the web server experienced blocking.

---

## Fix: gevent to gthread

### Worker Type Switch

We switched gunicorn's worker type from `gevent` to `gthread`.

`gthread` uses Python's native threading. No monkey patching means `concurrent.futures.ThreadPoolExecutor` and gRPC work correctly together.

After the switch, BigQuery query logic ran without blocking on the web server. SqlLab, chart development, and force refresh all worked normally.

### Concurrency Impact

Switching from gevent to gthread changes the concurrency characteristics. gevent uses greenlets, handling thousands of concurrent connections with minimal memory. gthread uses OS threads, so concurrent connections are limited by the thread pool size.

In our environment, Superset's concurrent user count doesn't reach thousands of simultaneous connections, so gthread was sufficient. With appropriate `--threads` configuration per worker, the practical concurrency difference is negligible.

### Upstream Contribution

Based on this experience, we submitted a PR to the official Superset documentation adding a warning against using the gevent worker type with BigQuery data sources. It was merged. Hopefully it saves others from wasting time on the same issue.

We also shared the async mode workaround and the gthread root cause fix in the community issue thread.

---

## Note on Impersonation

An additional consideration for BigQuery integration is per-user access control (impersonation). If all queries run through a single service account, granular permission control is difficult. Running queries under individual users' GCP accounts requires impersonation support.

The Superset community has active discussions around BigQuery impersonation and an OAuth2-based database authentication proposal (SIP-85). While not fully implemented yet, once this feature stabilizes, per-user BigQuery access control through Superset becomes viable.

---

## Takeaways

**When the same query produces different results depending on the execution path, suspect the execution environment.** SqlLab runs on Celery workers; charts run on the gunicorn web server. That difference was everything. The assumption "same query, same result" can lead your debugging in the wrong direction.

**gevent's monkey patching is a double-edged sword.** The very mechanism that gives gevent high concurrency — monkey patching Python's standard library — causes conflicts with libraries that depend on native threads, like gRPC. When using gevent in the Python ecosystem, verify that none of your dependencies rely on gRPC or native threading.

**Worker type selection is constrained by your dependencies.** Choosing a gunicorn worker type isn't just about concurrency requirements. You also need to check compatibility with the threading model of your application's libraries. BigQuery SDK + gRPC + gevent is structurally incompatible.

**Read community issue threads to the end.** A comment in the Superset issue thread saying "switching from gunicorn to waitress WSGI fixed it" was the pivotal clue that pointed us toward gevent as the problem. The habit of reading through the latest comments in issue threads matters.

**When you solve a problem, contribute upstream.** Even a one-line documentation PR saves time for the next person hitting the same issue. The cost of contributing is small relative to the value it provides to the community.

**References:**
- [Superset: Unable to query BigQuery data](https://github.com/apache/superset/issues/23979)
- [python-bigquery: to_dataframe blocked with gevent](https://github.com/googleapis/python-bigquery/issues/697)
- [gevent: BigQuery to_dataframe blocking issue](https://github.com/gevent/gevent/issues/1797)
- [gRPC: gevent compatibility](https://github.com/grpc/grpc/issues/4629)
- [Superset docs: add notice not to use gevent with BigQuery](https://github.com/apache/superset/pull/24564)
- [Superset: BigQuery impersonation discussion](https://github.com/apache/superset/discussions/18269)
- [Superset: OAuth2 for databases (SIP-85)](https://github.com/apache/superset/issues/20300)
- [Redash: BigQuery does not show schema](https://discuss.redash.io/t/big-query-does-not-show-schema/10649)
- [Gunicorn: Design / Server Model](https://gunicorn.org/design/)
