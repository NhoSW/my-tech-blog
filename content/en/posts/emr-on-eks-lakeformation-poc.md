---
title: "Adding Access Control to EMR-on-EKS Spark Jobs: LakeFormation PoC Through 10 Issues"
date: 2026-03-03
draft: false
categories: [Data Engineering]
tags: [emr, eks, lakeformation, spark, ranger, aws, access-control, iceberg]
showTableOfContents: true
summary: "We needed to add job-level access control to EMR-on-EKS Spark jobs. Ranger was ruled out due to EMR-on-EKS's structural limitations — no master node, no plugin installation path. We chose LakeFormation, and hit 10 issues during PoC: service label selector mismatches, FGAC blocking RDD operations/UDFs/synthetic types, cross-account Glue restrictions, and more. Here's how we identified each cause and found workarounds."
---

We needed access control for Spark jobs running on EMR-on-EKS. EMR-on-EC2 was only used by our data services team, so strict access control wasn't critical there. But EMR-on-EKS was used by multiple teams. We needed to restrict which tables each team could access.

Three options were on the table: Apache Ranger, AWS LakeFormation, and Databricks Unity Catalog. Unity Catalog was immediately ruled out — it couldn't control write operations on Iceberg format tables. After evaluating Ranger and LakeFormation, we concluded that Ranger was structurally incompatible with EMR-on-EKS. We proceeded with LakeFormation, and encountered 10 issues along the way.

---

## Why Ranger Doesn't Work on EMR-on-EKS

Understanding Ranger's architecture on EMR-on-EC2 is key.

On EMR-on-EC2, Ranger depends on the master node. When an EMR cluster launches, a Kerberos KDC server is automatically set up on the master node. The EMR Record Server is installed there. EMR Spark Ranger plugin and EMRFS S3 Ranger plugin are also installed on the master node. When Spark accesses data, the Record Server queries Ranger policies to verify permissions.

EMR-on-EKS has no master node. Spark driver and executor run as K8s pods. There's nowhere to deploy the Record Server, nowhere to install Ranger plugins. The Kerberos KDC server would need to be built from scratch. What's a checkbox option on EMR-on-EC2 becomes a full infrastructure buildout on EMR-on-EKS.

The definitive signal: EMR-on-EKS official documentation contains no mention of Ranger integration whatsoever. EMR-on-EC2 documentation has detailed sections on Ranger architecture, components, and considerations. AWS itself doesn't support Ranger on EMR-on-EKS.

---

## LakeFormation and EMR-on-EKS: The Dual Namespace Architecture

LakeFormation is officially supported on EMR-on-EKS starting from v7.7. When enabled, Spark job execution architecture changes fundamentally.

In a normal EMR-on-EKS job, driver and executor pods run in a single namespace. With LakeFormation enabled, pods are created in two namespaces — **user namespace** and **system namespace**. User-submitted code runs in the user namespace. The system namespace driver handles actual S3 access and LakeFormation permission verification. This separation prevents user code from directly accessing S3.

We set up a PoC environment: an EMR 7.8 virtual cluster with LakeFormation security configuration, job submission through Airflow.

Jobs submitted successfully. Then 10 issues appeared in succession.

---

## Issue 1: Cross-Account Glue Catalog Access Blocked

```
org.apache.spark.SparkUnsupportedOperationException:
Cross account access with fine-grained access control is only supported
with AWS Resource Access Manager.
```

Previously, we accessed another account's Glue catalog through IAM role permissions attached to pods. With LakeFormation enabled, this path is blocked. Cross-account access requires AWS Resource Access Manager (RAM).

For the PoC, we worked around this by using the same account's Glue catalog. RAM configuration was deferred as a separate work item.

---

## Issue 2: Service Label Selector Mismatch

When a job is submitted, driver pods and headless services are created in both user and system namespaces. The system namespace driver tried to connect to the user namespace driver's service and got `UnknownHostException`.

```
Failed to resolve name. status=Status{code=UNAVAILABLE,
description=Unable to resolve host
spark-000000035dnfu300a2b-driver-svc.dataplatform-emr-beta-780-lf.svc}
```

The service existed. The problem was that no endpoints were attached to it.

We investigated. The headless service's label selector included a `spark-app-name` condition. EMR's auto-generated services expected `spark-app-name: spark-000000035ho5ophkc5n`, but we were overriding the Spark app name to include Airflow task names for monitoring. The actual pod's `spark-app-name` label didn't match the service selector.

```yaml
# Service selector expects
spark-app-name: spark-000000035ho5ophkc5n

# Actual pod label (overridden)
spark-app-name: beta-airflow-spark-to-spark-container-iceberg-test-v1-spark-to
```

Notably, a separate label key `spark-app-selector` already carried the same identifier, making the `spark-app-name` selector redundant. We opened an AWS support case requesting removal of this selector condition.

Temporary fix: removed the `--name` parameter from `sparkSubmitParameters` to prevent app name override.

---

## Issues 3–4: FGAC Security Restrictions — UDFs and Function Creation

LakeFormation adds a security layer called Fine-Grained Access Control (FGAC) to EMR's Spark distribution. Implemented in the `org.apache.spark.fgac` package, it intercepts query execution plans to enforce column/row-level permission filtering.

The problem: FGAC is extremely conservative.

**Synthetic type blocking:** With FGAC enabled, synthetic types generated by the Scala compiler during compilation can't be serialized.

```
java.lang.IllegalArgumentException: Cannot parse synthetic types.
```

Classes that don't exist in static code but are internally generated by the compiler trigger FGAC's security validation. Identically-defined UDFs behaved differently — some failed, some didn't — likely due to synthetic types in imported utility modules.

**CREATE FUNCTION blocking:** FGAC doesn't support the `CREATE FUNCTION` SQL statement at all.

```
java.lang.UnsupportedOperationException:
Spark FGAC: CREATE FUNCTION command is not supported
```

Both issues were worked around by temporarily removing the affected UDFs.

---

## Issues 5–6: Configuration Key Confusion

Spark configuration keys required for LakeFormation integration differ subtly from existing keys.

**Region setting:** FGAC's LakeFormation client requires `spark.sql.catalog.iceberg.client.region`. Without it, initialization fails. The EMR-on-EKS troubleshooting docs covered this one.

**Glue account ID:** We previously used `spark.sql.catalog.iceberg.glue.id` to specify the Glue catalog account. But the FGAC module references a different key: `spark.sql.catalog.iceberg.glue.account-id`. We'd already removed `glue.id` for Issue 1, but the separate `glue.account-id` setting triggered the same cross-account error again.

---

## Issue 7: System Namespace Logs Are the Key

The user namespace driver shows only this:

```
org.apache.spark.fgac.error.SparkFGACException:
Spark FGAC experienced an internal error(15b27496-...) while executing this request
```

Impossible to diagnose. But checking the system namespace driver logs reveals the actual cause:

```
org.apache.iceberg.exceptions.NotFoundException:
Location does not exist: s3://woowa-ds-emrfs-beta-hive/dstemp_beta/
log_baemin_app_ib/metadata/00005-8d4df03f-...metadata.json
```

The test table's metadata file had been deleted. Creating a fresh table resolved it.

An important discovery came from this debugging: system namespace driver logs confirmed LakeFormation's column-level access control was actually working:

```
INFO AWSLakeFormationAccessControlUtils:
  adding column row filter for table log_baemin_app_ib: [viewingtime, TRUE]
INFO AWSLakeFormationAccessControlUtils:
  adding combined row filter for table `627051304531`.`dstemp_beta`.`log_baemin_app_ib`: TRUE
```

For FGAC troubleshooting, always check the system namespace driver logs, not the user namespace.

---

## Issues 8–9: FGAC Blocks Spark Features

FGAC blocks Spark features that could bypass permission filtering.

**RDD operations blocked:** FGAC enforces column/row-level permission filtering at the Spark SQL level. Dropping to the RDD level would bypass these filters, so RDD operations are entirely blocked.

```
org.apache.spark.fgac.error.SparkRDDUnsupportedException:
RDD execution is not supported
```

Existing code that called `df.rdd.partitions.size` to check partition counts was caught by this restriction.

**SPARK_PARTITION_ID blocked:** Calls to `SPARK_PARTITION_ID()` are also forbidden. This function exposes physical execution information that could be exploited for partition routing or filtering, potentially defeating row-level security.

```
org.apache.spark.fgac.error.SparkSecurityValidationException:
Disallowed class cannot be resolved from the serialization.
(org.apache.spark.sql.catalyst.expressions.SparkPartitionID)
```

Both issues were worked around by removing the affected code sections.

---

## Issue 10: DataFrameWriter.insertInto Incompatibility

The final issue hit during the Iceberg table write step.

```
org.apache.spark.fgac.serialization.ModelTransformationException:
No transformation available for type
'class org.apache.iceberg.spark.source.SparkTable'.
```

When `DataFrameWriter.insertInto()` is used, Spark internally creates an insert logical plan containing a `SparkTable` object. FGAC's serialization logic doesn't support this type.

Switching to `DataFrameWriterV2` routes through Iceberg's V2 write path directly, eliminating the need for internal representations like `SparkTable` and bypassing FGAC serialization.

This was the last issue. After 10 rounds of troubleshooting, the test Spark job completed successfully.

---

## FGAC's Impact on Existing Code

Through the 10 issues, FGAC's characteristics became clear. The `org.apache.spark.fgac` package that AWS added to EMR's Spark distribution conservatively restricts many standard Spark features for security.

| Restriction | Reason | Impact |
|-------------|--------|--------|
| RDD operations | Can bypass permission filters | `df.rdd.*` calls blocked |
| CREATE FUNCTION | Arbitrary function creation | SQL UDF registration blocked |
| Synthetic types | Can't be security-validated | Some Scala UDFs fail |
| SPARK_PARTITION_ID | Row-level security bypass | Partition info queries blocked |
| DataFrameWriter.insertInto | SparkTable serialization unsupported | Must switch to V2 API |
| Cross-account Glue | RAM required | Direct IAM access blocked |

These restrictions don't just affect internal batch pipelines. They affect Spark code submitted by other teams, custom JVM packages, and SQL queries. Compatibility verification with each team's Spark code is essential before production deployment.

---

## Rollout Strategy

Based on the PoC results, a phased rollout was the realistic approach.

Existing Airflow EMR/Spark connections remain without LakeFormation. A new set of LakeFormation-enabled EMR/Spark connections is provided separately. Users verify their pipelines work in the LakeFormation environment before migrating. Opt-in, not forced migration, minimizes risk to existing pipelines.

---

## The Alternative We Considered: Pre-Submission Permission Checks

As a fallback if LakeFormation proved unworkable, we evaluated a different approach: checking permissions at the query submission layer rather than at the EMR/Spark engine level.

Spark jobs are submitted from only three places: Airflow, Zeppelin, and Jupyter. At each submission point, parse the query, check source table permissions via Ranger API, and block submission entirely if unauthorized.

This approach has merits. Minimal additional infrastructure cost. No unnecessary pods spawn when permissions are denied. Existing Ranger policies for Trino can be directly reused. Compatible with the governance team's Ranger-based policy automation API.

But it requires implementing permission check logic separately at each submission point. DataFrame API calls that directly access S3 paths can't be controlled. Team feedback was that it felt "too artisanal." We decided to pursue LakeFormation further first.

---

## Takeaways

**"Ranger integration" on EMR-on-EC2 and EMR-on-EKS are completely different stories.** EMR-on-EC2's Ranger integration depends on the master node infrastructure. EMR-on-EKS has no master node, so that entire foundation is absent. They share the EMR name but have different architectural premises.

**FGAC's security philosophy is "allowlist."** Only known-safe patterns are permitted; everything else is blocked. RDD, synthetic types, SPARK_PARTITION_ID, CREATE FUNCTION — all blocked because they "could potentially bypass permission filters." How much existing Spark code hits these restrictions can only be discovered by running it.

**System namespace logs are the debugging key.** User namespace drivers output only `internal error`. Actual causes are in system namespace driver logs. LakeFormation job troubleshooting is impossible without checking these logs.

**A single label selector can stop everything.** Overriding Spark app names is common practice for monitoring and management. But when EMR's auto-generated headless service selectors depend on this value, service discovery breaks. Always verify the label structure of auto-generated resources.

**Feature restrictions are discovered through experimentation, not documentation.** FGAC's restrictions on Spark features aren't fully documented. Most of our 10 issues were discovered by actually running jobs. Had we applied this to production without a PoC, every team's pipeline would have failed simultaneously.

**References:**
- [EMR: Integrate with Apache Ranger](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-ranger.html)
- [EMR: Ranger Architecture](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-ranger-architecture.html)
- [EMR: Ranger Components](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-ranger-components.html)
- [EMR-on-EKS: FGAC Troubleshooting](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/security_iam_fgac-troubleshooting.html)
- [AWS Lake Formation: Getting Started](https://docs.aws.amazon.com/lake-formation/latest/dg/getting-started-setup.html)
