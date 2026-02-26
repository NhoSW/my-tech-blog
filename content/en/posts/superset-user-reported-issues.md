---
title: "Superset: Resolving Six User-Reported Issues"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [superset, csv, encoding, postgresql, troubleshooting]
showTableOfContents: true
summary: "Our data governance team reported six issues they found while using Superset. From a missing .png extension on Slack screenshots to a CSV download 500 error caused by PostgreSQL's idle_in_transaction_session_timeout killing the metadata DB connection mid-download. The trickiest one required tracing through RDS error logs to find that a 60-second timeout on the metadata database was cutting off large CSV exports."
---

Our data governance team actively adopted Superset and shared six issues they encountered during real-world usage. These ranged from simple configuration changes to tracing RDS error logs for root cause analysis. Here's the breakdown of each issue and how we resolved it.

---

## 1. Slack Chart Screenshot Missing File Extension

**Report:** Chart screenshots sent to Slack from Superset don't have a file extension. When users download the file, the OS doesn't recognize it as an image.

**Cause:** The Slack API call wasn't including a file extension in the filename parameter. Slack uses the filename as-is, so no extension in the API call means no extension on the downloaded file.

**Fix:** Updated the Slack API call to include `.png` in the filename.

---

## 2. Dashboard Auto-Refresh Interval Resets

**Report:** Setting an auto-refresh interval on a dashboard resets when the user reconnects.

**Cause:** The auto-refresh menu in the top-right corner of a dashboard is a session-only setting. By design, Superset doesn't persist this value server-side.

**Fix:** Guided users to set the refresh interval at the dashboard level through edit mode. Dashboard-level refresh intervals are stored in dashboard metadata and apply consistently for all users.

This wasn't a bug — it was a UX confusion. Having the same purpose served by two settings at different scopes (session vs. dashboard) is inherently confusing.

---

## 3. SqlLab Dataset Overwrite Bug

**Report:** Selecting "overwrite existing dataset" in SqlLab doesn't properly display the list of existing datasets.

**Cause:** A known upstream bug in Apache Superset.

**Fix:** Found the fix commit in the Superset community and cherry-picked it into our fork.

---

## 4. Korean Characters Garbled in CSV on Windows

**Report:** CSV files downloaded from Superset show garbled Korean characters when opened in Excel on Windows.

**Cause:** Superset was exporting CSVs in standard UTF-8. The problem is Microsoft Excel. When Excel opens a CSV without a BOM (Byte Order Mark), it falls back to the system's default encoding — which on Korean Windows is EUC-KR.

Even though the file is valid UTF-8, Excel has no way to detect this without the BOM. This isn't an issue on macOS or Linux, but in environments where Windows Excel users are common, it must be addressed.

**Fix:** Changed the CSV encoding from `utf-8` to `utf-8-sig`. Python's `utf-8-sig` encoding inserts a BOM (`EF BB BF`) at the beginning of the file. With the BOM present, Excel correctly interprets the file as UTF-8.

---

## 5. Query Result Row Limit: 100K to 1M

**Report:** Query results in Superset are limited to 100,000 rows. Users wanted up to 1 million rows.

**Fix:** Adjusted the `ROW_LIMIT` configuration to allow up to 1 million rows.

A simple configuration change, but raising the row limit isn't always the right call. Higher row counts mean proportionally higher memory usage and response times on the Superset web server. A million rows can consume several GB of memory depending on data width. This needs to be tuned based on the production environment's resource headroom.

---

## 6. CSV Download 500 Error — The Trickiest Issue

**Report:** Downloading query results as CSV with more than 500,000 rows fails with a 500 error after about one minute. Downloads with 200,000 rows work fine.

The specificity of this report made root cause analysis possible. "200K works, 500K doesn't" pointed to a time-based limit somewhere in the stack.

### Tracing the Error Logs

The Superset server logs at the time of failure showed:

```
psycopg2.OperationalError: SSL connection has been closed unexpectedly
```

At first glance, this looked like the Redshift connection dropping. But on closer inspection, this error was on the metadata database connection (Aurora PostgreSQL), not the Redshift data connection.

### Root Cause

Tracing the CSV download flow:

1. User requests a CSV download
2. Superset opens a transaction on the metadata DB (Aurora PostgreSQL) to check the user's download permissions
3. After the permission check, it starts fetching actual data from Redshift
4. With 500K+ rows, the Redshift fetch takes over one minute
5. During this time, the metadata DB transaction sits idle in an open state
6. Aurora RDS's `idle_in_transaction_session_timeout` is set to 60,000ms (1 minute), so it force-terminates the idle transaction
7. The terminated transaction causes Superset to return a 500 error

The Aurora RDS error logs confirmed this:

```
master@superset:[17810]:FATAL: terminating connection due to idle-in-transaction timeout
```

### Fix

We requested an RDS parameter group change: `idle_in_transaction_session_timeout` from 60,000ms (1 minute) to 600,000ms (10 minutes). Other timeout settings in the Superset stack (ALB, gunicorn) were already at 10 minutes, so aligning this value was the reasonable choice.

More elegant solutions exist — fixing the Superset code so the permission-check transaction doesn't stay open during data download, or placing a connection pooler like PgBouncer in front of the metadata DB. But the RDS parameter change was the fastest and safest fix, so we went with it.

After the parameter change, CSV downloads with 500K+ rows completed successfully.

---

## Bonus: Helm Chart db-init Job on ARM Nodes

The db-init Job in the Superset base Helm chart used an amd64 image but had no node selector configured. When the Job happened to schedule on an ARM node, it failed. Upgrading the Helm chart from v0.7.7 to v0.8.10 resolved this.

---

## Takeaways

**"N rows works, M rows doesn't" signals a time-based limit.** It's not the row count itself — it's a timeout triggered by processing time. Without this lens, you'd spend all your time investigating the Redshift connection.

**The connection that errors and the connection that causes the error can be different.** The SSL connection error happened on the metadata DB, not Redshift. Always verify which connection the error log refers to.

**Timeout settings must be consistent across the entire stack.** If the ALB and gunicorn are set to 10 minutes but the DB is at 1 minute, the shortest timeout becomes the bottleneck. Weakest link in the chain.

**CSV + non-ASCII + Windows = BOM is required.** `utf-8-sig` is Python's simplest way to insert a BOM. If your users include Windows Excel users, apply it by default.

**User reports are the most valuable QA.** Three of these six issues would have been near-impossible to catch through internal testing alone. CSV garbling on Windows and download failures at 500K+ rows only surface through real usage patterns.

**References:**
- [Apache Superset - SqlLab overwrite dataset fix](https://github.com/apache/superset/pull/25679)
- [Apache Superset - Helm chart db-init fix](https://github.com/apache/superset/pull/23416)
- [PostgreSQL: idle_in_transaction_session_timeout](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-IDLE-IN-TRANSACTION-SESSION-TIMEOUT)
- [Python: utf-8-sig encoding](https://docs.python.org/3/library/codecs.html#module-encodings.utf_8_sig)
