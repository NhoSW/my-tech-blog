---
title: "Superset: Resolving Six User-Reported Issues"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [superset, csv, encoding, postgresql, troubleshooting]
showTableOfContents: true
summary: "Our data governance team reported six issues they found while using Superset. From a missing .png extension on Slack screenshots to a CSV download 500 error caused by PostgreSQL's idle_in_transaction_session_timeout killing the metadata DB connection mid-download. The trickiest one required tracing through both Superset server logs and RDS error logs to discover that the error connection and the causal connection were different — the SSL error was on the metadata DB, not Redshift."
---

Our data governance team had been actively using Superset and shared six issues they discovered. Thanks to their hands-on usage and reporting, we caught problems that our internal testing had missed.

The issues varied widely in difficulty. Some were one-line code fixes. Others required cross-referencing Superset server logs with RDS error logs to finally pinpoint the root cause. Here's the breakdown of each issue, its cause, and how we resolved it.

---

## 1. Slack Chart Screenshot Missing File Extension

**Report:** Chart screenshots sent from Superset to Slack don't have a file extension. When users try to download and open the file, the OS doesn't recognize it as an image and asks them to manually specify the format.

**Cause:** Superset was calling Slack's `files.upload` API without including an extension in the `filename` parameter. Slack uses the uploaded filename as-is — if the filename is `chart_screenshot` without an extension, that's exactly what Slack stores.

On macOS, the OS often infers file type from content, so extensionless image files may still open correctly. But on Windows, the OS relies on file extensions for type detection. The reporter was using Windows, which is how this issue surfaced.

**Fix:** Updated the Slack API call to include `.png` in the filename.

---

## 2. Dashboard Auto-Refresh Interval Resets

**Report:** After setting an auto-refresh interval on a dashboard, the setting disappears when the user closes the browser and comes back.

**Cause:** The auto-refresh dropdown in the top-right corner of a dashboard is a session-only setting. By design, Superset doesn't persist this value server-side. Closing the browser tab clears it.

The confusion stems from Superset having two settings that serve the same purpose at different scopes:

- **Session-level setting**: The auto-refresh dropdown in the dashboard header. Applies only to the current session; lost when the browser tab closes.
- **Dashboard-level setting**: The refresh interval configured in dashboard edit mode. Stored in dashboard metadata, applied consistently for all users.

The reporter was using the session-level setting without knowing the dashboard-level setting existed. This is arguably a UX design issue — having the same concept at two different scopes with different persistence behavior is inherently confusing.

**Fix:** Guided users to set the refresh interval through dashboard edit mode.

---

## 3. SqlLab Dataset Overwrite Bug

**Report:** When saving query results as a dataset in SqlLab and selecting "overwrite existing dataset," the list of existing datasets either shows empty or search doesn't work.

**Cause:** A known upstream bug in Apache Superset. The API endpoint that loads the dataset list had a filtering logic error.

When the report came in, we first searched Superset's GitHub Issues. The same issue had already been reported, and a fix PR had been merged. However, the fix hadn't yet reached the Superset version we were running.

**Fix:** Cherry-picked the fix commit from the upstream repository into our fork.

---

## 4. Korean Characters Garbled in CSV on Windows

**Report:** CSV files downloaded from Superset show garbled Korean characters when opened in Excel on Windows. Opening the same file in Notepad shows the text correctly.

**Cause:** Superset was exporting CSVs in standard UTF-8. The file is a perfectly valid UTF-8 document, and most text editors handle it correctly.

The problem is Microsoft Excel. When Excel opens a CSV, it checks for a BOM (Byte Order Mark) at the beginning of the file. The BOM is a special 3-byte marker (`EF BB BF`) at position zero that signals "this file is UTF-8." Without a BOM, Excel falls back to the system's default encoding — which on Korean Windows is EUC-KR (CP949).

When Excel interprets UTF-8-encoded Korean characters as EUC-KR, they garble. For example, the string "Seoul" in Korean ("서울") is `EC 84 9C EC 9A B8` in UTF-8, which maps to completely different characters in EUC-KR.

On macOS and Linux, the system default encoding is UTF-8, so missing BOMs cause no issues. But in environments with Windows Excel users, the BOM is essential.

**Fix:** Changed the CSV export encoding from `utf-8` to `utf-8-sig`. Python's `utf-8-sig` encoding automatically inserts the 3-byte BOM at the start of the file. The code change is a single parameter:

```python
# Before
df.to_csv(path, encoding='utf-8')

# After
df.to_csv(path, encoding='utf-8-sig')
```

With the BOM present, Excel correctly interprets the file as UTF-8, and Korean characters display properly.

A note: BOM is optional in the UTF-8 standard, and some Unix tools treat BOM-prefixed UTF-8 files as anomalous. But when the primary consumer of CSVs is Excel, including the BOM is the right default.

---

## 5. Query Result Row Limit: 100K to 1M

**Report:** Query results in Superset are limited to 100,000 rows. Users needed up to 1 million rows for their analysis workflows.

**Fix:** Adjusted the `ROW_LIMIT` configuration to allow up to 1 million rows.

A simple configuration change, but blindly raising the row limit isn't always the right call. Higher row counts mean proportionally higher memory usage and response times on the Superset web server. At a million rows, memory consumption can reach several GB depending on column count and data types.

Our environment had sufficient memory headroom on the Superset web server, so we raised it to 1 million. But this needs to be tuned based on each environment's resource situation. If memory OOMs start occurring, the limit may need to come back down.

Additionally, raising the row limit increases the likelihood of hitting the next issue (#6) — large CSV downloads timing out. These two issues are closely related, which is why we addressed them together.

---

## 6. CSV Download 500 Error — The Trickiest Issue

**Report:** Downloading query results as CSV with more than 500,000 rows fails with a 500 error after about one minute. Downloads with 200,000 rows work fine.

The report itself contained the debugging clue. "200K works, 500K doesn't" means this isn't a data size problem — it's a processing time problem. There's a time-based limit somewhere in the stack.

### Tracing the Error Logs

We started with the Superset server logs at the time of failure.

```
psycopg2.OperationalError: SSL connection has been closed unexpectedly
```

Our first instinct was that the Redshift connection had dropped. Superset needs to fetch data from Redshift for CSV downloads, and with 500K rows, a Redshift-side timeout seemed plausible.

But looking more carefully at the error, this `psycopg2` error wasn't on the Redshift connection. It was on the metadata database connection — Aurora PostgreSQL. Redshift uses the `sqlalchemy-redshift` driver; the metadata DB uses `psycopg2`.

Why would the metadata DB error during a data download?

### Root Cause

Tracing the CSV download flow at the code level:

1. User requests a CSV download
2. Superset opens a transaction on the metadata DB (Aurora PostgreSQL) to check the user's download permissions
3. Permission check passes; within the same request context, it starts fetching actual data from Redshift
4. With 500K+ rows, the Redshift fetch takes over one minute
5. During this time, the metadata DB transaction from step 2 sits idle — no queries executing, just an open transaction waiting
6. Aurora RDS's `idle_in_transaction_session_timeout` is set to 60,000ms (1 minute)
7. PostgreSQL force-terminates the idle transaction once it exceeds 1 minute
8. The terminated metadata DB connection causes Superset to return a 500 error

The key is step 5. The permission check was already complete, but the transaction remained open. This is due to Superset's SQLAlchemy session management, which maintains the transaction until the request completes.

The Aurora RDS error logs at the corresponding timestamp confirmed this:

```
master@superset:[17810]:FATAL: terminating connection due to idle-in-transaction timeout
```

Checking the RDS parameter group, `idle_in_transaction_session_timeout` was set to 60,000ms (1 minute). This matched the report exactly. 200K rows completes within one minute; 500K rows exceeds it.

### Fix

We requested an RDS parameter group change: `idle_in_transaction_session_timeout` from 60,000ms (1 minute) to 600,000ms (10 minutes). Other timeout settings in the Superset stack — ALB timeout, gunicorn worker timeout — were already set to 10 minutes (600 seconds), so aligning the metadata DB timeout to match was the rational choice.

We considered more fundamental solutions. The permission-check transaction doesn't need to stay open during data download, so modifying the Superset code to commit the transaction immediately after the permission check is one approach. Alternatively, placing a connection pooler like PgBouncer in front of the metadata DB with transaction pooling mode would reduce idle transactions altogether.

But both approaches require code or architecture changes with broader impact that would need thorough validation. The RDS parameter change was immediate, non-invasive, and safe. We applied it first and filed a separate ticket for the code-level improvement.

After the parameter change, CSV downloads with 500K+ rows completed successfully.

---

## Bonus: Helm Chart db-init Job on ARM Nodes

Separate from user-reported issues, we found an operational issue ourselves.

The db-init Job in the Superset base Helm chart used an amd64-only image. The problem: the Job's pod template had no node selector, so the Kubernetes scheduler could place it on an ARM (Graviton) node, causing an architecture mismatch failure.

Our EKS cluster uses a mix of amd64 and arm64 nodes for cost optimization. Without a node selector, amd64-only images fail probabilistically — the Job works sometimes and fails other times depending on which node the scheduler picks. This intermittent behavior made it take a while to identify the root cause.

Upgrading the Helm chart from v0.7.7 to v0.8.10 resolved the issue. The newer version applies architecture-appropriate node selectors to the db-init Job.

---

## Takeaways

**"N rows works, M rows doesn't" signals a time-based limit.** If the row count itself were the problem, it would fail at 100K rows too. When there's a specific row count boundary, suspect a timeout triggered by processing time. Without this perspective, you'll spend all your effort investigating the data source connection (Redshift) and miss the actual cause entirely.

**The connection that errors and the connection that causes the error can be different.** In this case, the SSL connection error occurred on the metadata DB (Aurora PostgreSQL), not Redshift. Checking the error message's driver (`psycopg2` vs `sqlalchemy-redshift`) was how we identified which connection failed. Jumping to "oh, Redshift timed out" based on the error message alone would have sent us in completely the wrong direction.

**Timeout settings must be consistent across the entire stack.** If the ALB timeout is 10 minutes, gunicorn worker timeout is 10 minutes, but the metadata DB idle transaction timeout is 1 minute, the chain breaks at its weakest link. In distributed systems, timeout configuration must be audited along the entire request path.

**CSV + non-ASCII + Windows = BOM is required.** `utf-8-sig` is Python's simplest way to insert a BOM — just a one-parameter change. If your users include Windows Excel users, apply it as the default for CSV exports. For macOS and Linux users, the BOM is harmless, making it the safe default.

**User reports are the most valuable QA.** At least three of these six issues (Windows CSV garbling, 500K+ row CSV download failure, Slack file extension) would have been extremely difficult to catch through internal testing. Our team primarily tests on macOS, and downloading 500K+ rows as CSV wasn't in our test scenarios. Real usage patterns differ from developer test scenarios. Having users who actively use the tool and report issues is genuinely invaluable.

**References:**
- [Apache Superset - SqlLab overwrite dataset fix](https://github.com/apache/superset/pull/25679)
- [Apache Superset - Helm chart db-init fix](https://github.com/apache/superset/pull/23416)
- [PostgreSQL: idle_in_transaction_session_timeout](https://www.postgresql.org/docs/current/runtime-config-client.html#GUC-IDLE-IN-TRANSACTION-SESSION-TIMEOUT)
- [Python: utf-8-sig encoding](https://docs.python.org/3/library/codecs.html#module-encodings.utf_8_sig)
