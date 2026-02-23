---
title: "Airflow 3.0 Migration Guide: Lessons from a Large-Scale DAG Environment"
date: 2026-02-23
draft: false
categories: [Data Engineering]
tags: [airflow, migration, orchestration, python, data-pipeline]
showTableOfContents: true
summary: "Practical lessons from migrating to Airflow 3.x ahead of the 2.x EOL. Covers major breaking changes, a phased upgrade strategy, DAG compatibility approaches, and hard-won lessons from operating hundreds of DAGs in production."
---

The End of Life for Airflow 2.x is approaching on April 22, 2026. Our team carried out an Airflow 3.x migration in a production environment running hundreds of DAGs. This post is a record of the breaking changes we encountered, the phased upgrade strategy we used, and the practical lessons we learned from migrating at scale.

---

## Why Migrate Now?

### Key Improvements in Airflow 3.x

Airflow 3.x is not just a major version bump. Fundamental changes have occurred at the architectural level.

- **DAG versioning**: No more hacking version suffixes onto `dag_id` or dealing with scheduling confusion when the schedule changes.
- **Native backfill**: Backfills that used to depend on the CLI or custom plugins are now supported directly from the web UI.
- **Event/asset-based triggers**: Scheduling options now go far beyond simple cron expressions.
- **React-based web UI**: The UI has been completely rebuilt from Flask App Builder to React, dramatically improving usability.

### Architectural Change: The Arrival of the API Server

The most significant architectural change in 3.x is that **the API Server has become the sole gateway to the metadata database.**

```
Airflow 2.x:
  Webserver ─── MetaDB
  Worker ────── MetaDB
  Scheduler ─── MetaDB
  DAG Code ──── MetaDB (direct access possible)

Airflow 3.x:
  API Server ── MetaDB (sole access path)
  Webserver ─── API Server
  Worker ────── API Server
  Scheduler ─── API Server
  DAG Code ──── API Server (no direct access)
```

As a result, **any pattern where DAG top-level code directly accessed the metadata database will break.** This is the single most impactful change in the entire migration.

---

## Phased Upgrade Strategy

Jumping straight to the latest version is risky. We devised a four-phase strategy.

### Phase 1: Update to the Latest 2.x Version (2.11) (Optional)

This serves as a safety net in case issues arise during the jump to 3.x. Version 2.11 displays deprecation warnings for features that will be removed in 3.x, so you can identify which code needs to be modified ahead of time.

### Phase 2: Update to 3.0.x

If you're on Python 3.9, only **3.0.x is supported**, not the latest 3.1.x. Upgrade the Airflow major version first, before upgrading Python.

### Phase 3: Upgrade Python (3.9 → 3.12+)

Airflow 3.1.x does not support Python 3.9. Aim for Python 3.12 or later, but compromise with 3.10 or 3.11 if dependency compatibility issues arise.

### Phase 4: Update to 3.1.x

Finally, upgrade to the latest stable release.

### Sequential Rollout Across Environments

```
DEV → BETA & Personal Environments → STAGE → PROD
```

Proceed to the next environment only after thorough validation in each one. We spent about two weeks validating in DEV and one week in BETA.

---

## Key Breaking Changes and How to Handle Them

### 1. `schedule_interval` → `schedule`

This is the most commonly encountered change. Simply pass the same cron expression you used for `schedule_interval` to `schedule` instead.

```python
# Before (Airflow 2.x)
DAG(
    dag_id="my_dag",
    schedule_interval="5 2 * * *",
)

# After (Airflow 3.x)
DAG(
    dag_id="my_dag",
    schedule="5 2 * * *",
)
```

It's a straightforward substitution, but when you have hundreds of DAGs, every single one must be updated without exception. We'll cover how to automate verification in CI later.

### 2. Passing Non-Existent Operator Arguments Is No Longer Allowed

In Airflow 3.x, individual tasks now receive serialized DAGs from the metadata database for execution. As a result, the `allow_illegal_arguments` setting has been removed, and **passing an argument not defined on the operator will cause the DAG import itself to fail.**

```python
# Code like this worked silently in 2.x, but raises an error in 3.x
MyOperator(
    task_id="my_task",
    num_partition=10,  # Actual argument name is num_partitions (plural)
)
```

```
TypeError: Invalid arguments were passed to MyOperator (task_id: my_task).
Invalid arguments were:
**kwargs: {'num_partition': 10}
```

This change actually serves as **an opportunity to catch latent bugs.** If a misspelled argument had been silently ignored for a long time, this migration is the moment to fix it.

### 3. Deprecated Context/Template Variables Removed

Variables that only showed deprecation warnings in 2.x have been completely removed in 3.x. The most impactful one is `execution_date`.

| Deprecated Variable | Replacement |
|---------------------|------------|
| `{{ execution_date }}` | `{{ logical_date }}` or `{{ data_interval_start }}` |
| `{{ next_execution_date }}` | `{{ data_interval_end }}` |
| `{{ prev_execution_date_success }}` | `{{ prev_data_interval_start_success }}` |

Both Jinja templates and Python code need to be updated.

```python
# Jinja templates
# Before
"SELECT * FROM table WHERE dt = '{{ execution_date }}'"
# After
"SELECT * FROM table WHERE dt = '{{ logical_date }}'"

# Python context
# Before
execution_date = context["execution_date"]
# After
logical_date = context["logical_date"]
```

### 4. Per-Database Operators Unified → `SQLExecuteQueryOperator`

Individual operators for MySQL, PostgreSQL, Trino, etc. have been consolidated into a single `SQLExecuteQueryOperator`. Internally, it automatically selects the appropriate Hook based on the connection type.

```python
# Before (Airflow 2.x)
from airflow.providers.mysql.operators.mysql import MySqlOperator
MySqlOperator(
    task_id="task",
    mysql_conn_id="my_conn",
    sql="SELECT 1"
)

# After (Airflow 3.x)
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
SQLExecuteQueryOperator(
    task_id="task",
    conn_id="my_conn",  # DB-specific conn_id → unified conn_id
    sql="SELECT 1"
)
```

### 5. `DummyOperator` → `EmptyOperator`

You need to use an import path that works in both 2.x and 3.x.

```python
# Works only in v2 (errors in 3.x)
from airflow.operators.dummy import DummyOperator

# Works only in v3
from airflow.providers.standard.operators.empty import EmptyOperator

# Compatible with both v2 & v3 (recommended)
from airflow.operators.empty import EmptyOperator
```

### 6. `SimpleHttpOperator` → `HttpOperator`

```python
# Before
from airflow.providers.http.operators.http import SimpleHttpOperator
# After
from airflow.providers.http.operators.http import HttpOperator
```

### 7. Connection Getter Methods → Direct Property Access

The Connection class interface has been changed to be more Pythonic.

```python
# Before
conn = BaseHook.get_connection("my_conn")
password = conn.get_password()
host = conn.get_host()

# After
conn = BaseHook.get_connection("my_conn")
password = conn.password
host = conn.host
```

### 8. Other Package Path Changes

```python
# cached_property
# Before: from airflow.compat.functools import cached_property
# After: from functools import cached_property (Python built-in)

# KubernetesPodOperator
# Before: from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import ...
# After: from airflow.providers.cncf.kubernetes.operators.pod import ...
```

---

## Migration Strategies for Large-Scale DAG Environments

### Add v3 Compatibility Checks to Your CI Pipeline

It's impossible to manually verify hundreds of DAGs. We added a CI job that automatically checks v3 compatibility at the MR (Merge Request) stage.

```yaml
# .gitlab-ci.yml example
airflow-v3-compat-check:
  stage: test
  image: apache/airflow:3.0.6-python3.12
  script:
    - pip install -r requirements.txt
    - python -m py_compile dags/**/*.py
    - airflow dags list --output table
  allow_failure: true  # Start as warning-only, then switch to required
```

Start with `allow_failure: true` to get a picture of the current state, then switch to mandatory checks as the migration deadline approaches.

### Intentionally Keep the Migration Hurdle High

This was the most important lesson we learned.

We could have applied a blanket compatibility patch to all DAG code. But we **deliberately chose to maintain the migration difficulty.** The reason was clear:

> A significant number of DAGs were being operated out of inertia but weren't actually in use.

By treating the migration as an opportunity, we nudged DAG owners into asking themselves **"Do we really need this DAG?"** As a result, a substantial number of unnecessary DAGs were cleaned up, which directly reduced operational overhead.

The specific process:

1. Compile a list of inactive DAGs and organize them in a shared spreadsheet
2. Notify DAG owners/departments and ask them to confirm whether each DAG is still needed by a deadline
3. Deactivate DAGs with no response by the deadline
4. DAG owners perform v3 compatibility patches themselves

### Proactively Update Custom Provider Packages

If you maintain in-house custom operators or utilities as a Provider package, the key is to **prepare a compatibility layer that absorbs Airflow core's breaking changes** ahead of time.

We incrementally updated our custom Provider package across four releases:

- v3.0.0: Basic compatibility
- v3.0.1: Operator argument validation support
- v3.0.2: Compatibility layer for deprecated context variables
- v3.0.3: Documentation and minor bug fixes

The approach was to minimize changes to user-facing code while handling v2/v3 branching logic internally within the Provider package.

### Helm Chart Updates

If you run Airflow on Kubernetes, the Helm chart needs to be updated as well. This is because the DAG Processor component and the API Server separation introduced in 3.x must be reflected.

A safe two-step approach is to first verify compatibility with the existing chart version, then update to the latest stable version once things have stabilized.

---

## FAB Auth Manager Issue

With the full web UI rebuild to React in 3.x, **the existing Flask App Builder (FAB)-based Auth Manager was removed from the default package.** If you're using a custom Security Manager, you'll need to install the package separately and update your code.

```
Failed to import WoowaSecurityManager, using default security manager
```

If you see this error, you need to explicitly install the FAB Auth Manager package and update the import paths.

---



## Closing Thoughts

Migrating to Airflow 3.x is not a simple version upgrade. It requires extensive work spanning architectural changes (API Server-centric design), code compatibility updates, and infrastructure modifications.

Here are the key lessons summarized:

1. **Upgrade in phases** -- Don't jump straight to the latest version. Follow the path: 2.11 → 3.0.x → Python upgrade → 3.1.x.
2. **Automate verification in CI** -- It's impossible for humans to manually check the compatibility of hundreds of DAGs.
3. **Treat the migration as a cleanup opportunity** -- Intentionally maintain the migration hurdle to naturally filter out unnecessary DAGs.
4. **Proactively update custom Providers** -- A compatibility layer that minimizes changes to user code is the key.

Don't be complacent just because there's still time until the Airflow 2.x EOL. Migrations in large-scale environments take longer than expected. It's not too late to start now.

**References:**
- [Upgrading to Airflow 3 - Apache Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/stable/howto/upgrading-to-3.html)
- [Apache Airflow 3 is Generally Available!](https://airflow.apache.org/blog/airflow-three-point-zero-is-here/)
- [Airflow 3.x Release Notes](https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html)
