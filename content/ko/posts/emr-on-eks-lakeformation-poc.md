---
title: "EMR-on-EKS Spark 잡에 LakeFormation 권한제어 도입: 10개의 이슈를 넘어서"
date: 2026-03-03
draft: false
categories: [Data Engineering]
tags: [emr, eks, lakeformation, spark, ranger, aws, access-control, iceberg]
showTableOfContents: true
summary: "EMR-on-EKS에 Spark 잡 레벨의 권한제어를 도입해야 했다. Ranger는 EMR-on-EKS의 구조적 한계(마스터 노드 부재, 플러그인 설치 불가)로 기각되었고, LakeFormation을 선택했다. PoC 과정에서 서비스 라벨 셀렉터 불일치, FGAC의 RDD/UDF/synthetic type 차단, cross-account Glue 접근 제한 등 10개의 이슈를 만났다. 각각의 원인 파악과 우회 방법, 그리고 FGAC가 기존 Spark 코드에 미치는 영향을 정리한다."
---

EMR-on-EKS에서 실행되는 Spark 잡에 권한제어를 도입해야 했다. EMR-on-EC2는 데이터서비스실만 사용하므로 엄격한 권한 관리가 덜 필요하지만, EMR-on-EKS는 다른 팀들도 사용한다. 팀별로 접근 가능한 테이블을 제한해야 하는 상황이었다.

선택지는 세 가지였다. Apache Ranger, AWS LakeFormation, Databricks Unity Catalog. Unity Catalog는 Iceberg 포맷에 대한 쓰기 작업 권한제어가 불가능해서 바로 제외되었다. Ranger와 LakeFormation을 검토한 결과, Ranger는 EMR-on-EKS에 구조적으로 연동이 어렵다는 결론에 도달했다. LakeFormation으로 PoC를 진행했고, 10개의 이슈를 만났다.

---

## Ranger가 EMR-on-EKS에서 안 되는 이유

EMR-on-EC2에서의 Ranger 연동 구조를 먼저 이해해야 한다.

EMR-on-EC2에서 Ranger는 마스터 노드에 의존한다. EMR 클러스터를 시작할 때 마스터 노드에 Kerberos KDC 서버가 자동으로 셋업되고, EMR Record Server가 설치된다. EMR Spark Ranger 플러그인과 EMRFS S3 Ranger 플러그인도 마스터 노드에 설치된다. Spark가 데이터에 접근할 때 Record Server가 Ranger 정책을 조회해서 권한을 검증하는 구조다.

EMR-on-EKS에는 마스터 노드가 없다. Spark driver와 executor가 K8s 파드로 실행될 뿐이다. Record Server를 배치할 곳이 없고, Ranger 플러그인을 설치할 곳도 없다. Kerberos KDC 서버도 별도로 구성해야 한다. EMR-on-EC2에서는 클러스터 시작 옵션으로 체크만 하면 되는 것을, EMR-on-EKS에서는 전부 직접 구축해야 한다.

결정적으로, EMR-on-EKS 공식 문서에는 Ranger 연동 관련 항목이 아예 존재하지 않는다. EMR-on-EC2 문서에는 Ranger 아키텍처, 컴포넌트, 고려사항까지 상세하게 기술되어 있는 것과 대조적이다. AWS 자체가 EMR-on-EKS에서의 Ranger를 지원하지 않는 것으로 판단했다.

---

## LakeFormation과 EMR-on-EKS: 이중 네임스페이스 아키텍처

LakeFormation은 EMR-on-EKS v7.7부터 공식 지원된다. 연동을 활성화하면 Spark 잡의 실행 구조가 바뀐다.

일반적인 EMR-on-EKS 잡에서는 하나의 네임스페이스에 driver와 executor 파드가 뜬다. LakeFormation이 활성화되면 **유저 네임스페이스**와 **시스템 네임스페이스** 두 곳에 각각 driver/executor 파드가 생성된다. 유저 네임스페이스에는 사용자가 제출한 코드가 실행되고, 시스템 네임스페이스의 driver가 실제 S3 접근과 LakeFormation 권한 검증을 수행한다. 사용자 코드가 직접 S3에 접근하지 못하도록 분리한 것이다.

이 구조를 확인하기 위해 먼저 PoC 환경을 구축했다. EMR 7.8 버전으로 LakeFormation 연동 security configuration을 주입한 가상 클러스터를 생성하고, Airflow에서 잡을 제출했다.

잡 자체는 제출에 성공했다. 하지만 그 뒤로 10개의 이슈가 연달아 나타났다.

---

## 이슈 1: Cross-Account Glue 카탈로그 접근 불가

```
org.apache.spark.SparkUnsupportedOperationException:
Cross account access with fine-grained access control is only supported
with AWS Resource Access Manager.
```

기존에는 파드에 부여된 IAM 롤의 권한으로 다른 계정의 Glue 카탈로그에 접근했다. LakeFormation이 활성화되면 이 방식이 차단된다. Cross-account 접근은 AWS Resource Access Manager(RAM)를 통해서만 허용된다.

PoC 단계에서는 동일 계정의 Glue 카탈로그를 사용하는 것으로 우회했다. RAM 설정은 별도 작업 범위로 분리했다.

---

## 이슈 2: 서비스 라벨 셀렉터 불일치

잡을 제출하면 유저/시스템 네임스페이스 양쪽에 driver 파드와 headless service가 생성된다. 그런데 시스템 네임스페이스의 driver가 유저 네임스페이스의 driver 서비스에 연결을 시도하면서 `UnknownHostException`이 발생했다.

```
Failed to resolve name. status=Status{code=UNAVAILABLE,
description=Unable to resolve host
spark-000000035dnfu300a2b-driver-svc.dataplatform-emr-beta-780-lf.svc}
```

서비스는 존재한다. 엔드포인트가 연결되지 않은 것이 문제였다.

원인을 파고들었다. headless service의 라벨 셀렉터에 `spark-app-name`이라는 조건이 포함되어 있었다. EMR이 자동 생성하는 서비스는 `spark-app-name: spark-000000035ho5ophkc5n` 같은 기본값을 셀렉터에 사용하는데, 우리는 Spark 앱 이름을 Airflow 태스크명을 포함한 형태로 오버라이드하고 있었다. 실제 파드의 `spark-app-name` 라벨 값이 서비스 셀렉터와 일치하지 않아서 엔드포인트가 연결되지 않은 것이다.

```yaml
# 서비스 셀렉터가 기대하는 값
spark-app-name: spark-000000035ho5ophkc5n

# 실제 파드에 붙은 값 (오버라이드됨)
spark-app-name: beta-airflow-spark-to-spark-container-iceberg-test-v1-spark-to
```

흥미로운 점은, `spark-app-selector`라는 다른 라벨 키가 이미 동일한 식별자를 담고 있어서 `spark-app-name` 셀렉터는 사실상 중복이라는 것이다. AWS 서포트 케이스를 열어 이 셀렉터 조건의 제거를 요청했다.

임시 해결책으로는 `sparkSubmitParameters`에서 `--name` 설정을 제거해서 앱 이름이 오버라이드되지 않도록 했다.

---

## 이슈 3~4: FGAC의 보안 제한 — UDF와 함수 생성

LakeFormation은 Spark의 EMR 배포판에 Fine-Grained Access Control(FGAC)이라는 보안 레이어를 추가한다. `org.apache.spark.fgac` 패키지로 구현되어 있으며, 쿼리 실행 계획을 가로채서 컬럼/로우 단위 권한 필터링을 강제한다.

문제는 이 FGAC가 매우 보수적이라는 것이다.

**Synthetic type 차단:** FGAC 활성화 시 Scala 컴파일러가 컴파일 과정에서 생성하는 synthetic type을 직렬화하지 못한다.

```
java.lang.IllegalArgumentException: Cannot parse synthetic types.
```

정적 코드에는 존재하지 않지만 컴파일러가 내부적으로 생성한 클래스가 FGAC의 보안 검증에 걸린다. 같은 문법으로 정의된 UDF임에도 어떤 것은 에러가 발생하고 어떤 것은 정상이었다. 해당 UDF가 임포트하는 유틸 모듈의 하위 코드에서 synthetic type이 생성되는 것으로 추정되지만, 에러 로그에 상세 정보가 없어서 정확한 원인은 파악하기 어려웠다.

**CREATE FUNCTION 차단:** FGAC는 `CREATE FUNCTION` SQL 문 자체를 지원하지 않는다.

```
java.lang.UnsupportedOperationException:
Spark FGAC: CREATE FUNCTION command is not supported
```

두 이슈 모두 해당 UDF들을 임시로 제거해서 우회했다.

---

## 이슈 5~6: 설정 키 혼동

LakeFormation 연동에 필요한 Spark 설정 키가 기존 설정 키와 미묘하게 달라서 혼란이 있었다.

**리전 설정:** FGAC의 LakeFormation 클라이언트는 `spark.sql.catalog.iceberg.client.region` 설정이 필요하다. 설정하지 않으면 초기화에 실패한다. 다행히 EMR-on-EKS 트러블슈팅 문서에 해당 내용이 있었다.

**Glue 계정 ID:** 기존에는 `spark.sql.catalog.iceberg.glue.id`로 Glue 카탈로그 계정을 지정했다. 그런데 FGAC 모듈은 이것과 다른 `spark.sql.catalog.iceberg.glue.account-id`를 참조한다. 이슈 1에서 `glue.id`를 제거하는 것으로 조치했는데, `glue.account-id`라는 별도 설정이 남아 있어서 동일한 cross-account 에러가 다시 발생했다.

---

## 이슈 7: 시스템 네임스페이스 로그의 중요성

유저 네임스페이스의 driver에서는 이런 에러만 보인다:

```
org.apache.spark.fgac.error.SparkFGACException:
Spark FGAC experienced an internal error(15b27496-...) while executing this request
```

원인 파악이 불가능하다. 하지만 시스템 네임스페이스의 driver 로그를 확인하면 실제 원인이 드러난다:

```
org.apache.iceberg.exceptions.NotFoundException:
Location does not exist: s3://woowa-ds-emrfs-beta-hive/dstemp_beta/
log_baemin_app_ib/metadata/00005-8d4df03f-...metadata.json
```

테스트 대상으로 삼은 오래된 테이블의 메타데이터 파일이 삭제되어 있었다. 테이블을 새로 생성해서 해결했다.

이 과정에서 한 가지 중요한 사실도 확인했다. 시스템 네임스페이스의 driver 로그에 LakeFormation의 컬럼 레벨 접근제어가 실제로 동작하고 있는 것이 기록되어 있었다:

```
INFO AWSLakeFormationAccessControlUtils:
  adding column row filter for table log_baemin_app_ib: [viewingtime, TRUE]
INFO AWSLakeFormationAccessControlUtils:
  adding combined row filter for table `627051304531`.`dstemp_beta`.`log_baemin_app_ib`: TRUE
```

FGAC 관련 디버깅에서는 유저 네임스페이스가 아니라 시스템 네임스페이스의 로그를 봐야 한다.

---

## 이슈 8~9: FGAC의 Spark 기능 차단

FGAC는 권한 필터링을 우회할 수 있는 Spark 기능들을 차단한다.

**RDD 연산 차단:** FGAC는 쿼리 수준에서 컬럼/로우 단위 권한 필터링을 Spark SQL 레벨에서 강제한다. RDD 레벨로 내려가면 이 필터를 우회할 수 있으므로, RDD 연산 자체가 차단된다.

```
org.apache.spark.fgac.error.SparkRDDUnsupportedException:
RDD execution is not supported
```

기존 코드에서 파티션 수를 확인하기 위해 `df.rdd.partitions.size`를 호출하는 부분이 있었는데, 이것도 차단되었다.

**SPARK_PARTITION_ID 차단:** 물리적 실행 정보를 노출하는 `SPARK_PARTITION_ID()` 호출도 허용되지 않는다. 파티션 라우팅이나 필터링에 악용될 수 있고, row-level security를 무력화할 가능성이 있기 때문이다.

```
org.apache.spark.fgac.error.SparkSecurityValidationException:
Disallowed class cannot be resolved from the serialization.
(org.apache.spark.sql.catalyst.expressions.SparkPartitionID)
```

두 이슈 모두 해당 코드를 임시 제거해서 우회했다.

---

## 이슈 10: DataFrameWriter.insertInto 비호환

마지막 이슈는 Iceberg 테이블 쓰기 단계에서 발생했다.

```
org.apache.spark.fgac.serialization.ModelTransformationException:
No transformation available for type
'class org.apache.iceberg.spark.source.SparkTable'.
```

`DataFrameWriter.insertInto()`를 사용하면 Spark 내부적으로 insert logical plan에 `SparkTable` 객체가 포함된다. FGAC의 직렬화 로직이 이 타입을 지원하지 않았다.

`DataFrameWriterV2`로 전환하면 Iceberg provider를 통해 V2 write path를 직접 타게 되고, `SparkTable` 같은 내부 representation이 필요 없어져서 FGAC 직렬화를 우회할 수 있었다.

이것이 마지막 이슈였다. 10번의 트러블슈팅을 거쳐, 테스트 Spark 잡이 정상 완료되었다.

---

## FGAC가 기존 코드에 미치는 영향

10개의 이슈를 통과하면서 FGAC의 특성이 명확해졌다. AWS가 Spark의 EMR 배포판에 커스텀으로 추가한 `org.apache.spark.fgac` 패키지는, 보안을 위해 보수적으로 많은 기존 Spark 기능을 제한한다.

| 제한 항목 | 이유 | 영향 |
|----------|------|------|
| RDD 연산 | 권한 필터 우회 가능 | `df.rdd.*` 호출 불가 |
| CREATE FUNCTION | 임의 함수 생성 차단 | SQL UDF 등록 불가 |
| Synthetic type | 보안 검증 불가 | 일부 Scala UDF 실패 |
| SPARK_PARTITION_ID | row-level security 우회 | 파티션 정보 조회 불가 |
| DataFrameWriter.insertInto | SparkTable 직렬화 미지원 | V2 API로 전환 필요 |
| Cross-account Glue | RAM 필수 | 기존 IAM 직접 접근 차단 |

이 제한들은 내부 배치 파이프라인에만 영향을 주는 것이 아니다. 다른 팀이 직접 제출하는 Spark 코드, 커스텀 JVM 패키지, SQL 쿼리 모두에 영향을 준다. LakeFormation 연동을 프로덕션에 적용하기 전에, 각 팀의 Spark 코드와의 호환성 검증이 필수다.

---

## 도입 전략

PoC 결과를 바탕으로, 단계적 도입이 현실적이라고 판단했다.

기존 Airflow EMR/Spark 커넥션은 LakeFormation 없이 그대로 유지한다. LakeFormation이 연동된 신규 EMR/Spark 커넥션을 별도로 제공하고, 사용자가 직접 LakeFormation 환경에서 자기 파이프라인이 정상 동작하는지 확인한 후에 이관하게 하는 방식이다. 강제 전환이 아니라 opt-in 방식으로 진행해야 기존 파이프라인에 대한 리스크를 최소화할 수 있다.

---

## 대안으로 검토했던 것: 쿼리 제출 전 권한 체크

LakeFormation이 불가능할 경우를 대비해 다른 접근도 검토했다. EMR/Spark 단이 아니라 쿼리 제출처에서 권한을 체크하는 방식이다.

Spark 잡의 제출 경로는 Airflow, Zeppelin, Jupyter 세 곳뿐이다. 각 제출처에서 쿼리를 파싱하고, Ranger API로 소스 테이블에 대한 권한을 조회해서, 권한이 없으면 제출 자체를 차단하는 것이다.

이 방식의 장점은 있다. 추가 인프라 비용이 최소화되고, 권한이 없으면 파드가 아예 뜨지 않으므로 리소스 낭비도 없다. 기존 Ranger의 Trino 정책 세트를 그대로 재활용할 수도 있다. 거버넌스팀이 구축 중인 Ranger 기반 정책 자동화 API와도 호환된다.

하지만 각 제출처마다 권한 체크 로직을 개별적으로 구현해야 하고, DataFrame API로 S3 경로에 직접 접근하는 경우는 제어할 수 없다. 팀 내 논의에서 "가내수공업 느낌"이라는 피드백이 나왔다. LakeFormation을 우선 더 검토하기로 했다.

---

## 배운 것

**같은 "Ranger 연동"이라도 EMR-on-EC2와 EMR-on-EKS는 완전히 다른 이야기다.** EMR-on-EC2에서의 Ranger 연동은 마스터 노드라는 인프라에 의존한다. 마스터 노드가 없는 EMR-on-EKS에서는 그 인프라가 통째로 빠진다. 같은 EMR이라는 이름을 공유하지만 아키텍처적 전제가 다르다.

**FGAC의 보안 철학은 "허용 목록"이다.** 알려진 안전한 패턴만 허용하고 나머지는 차단한다. RDD, synthetic type, SPARK_PARTITION_ID, CREATE FUNCTION 모두 "잠재적으로 권한 필터를 우회할 수 있다"는 이유로 차단된다. 기존 Spark 코드가 얼마나 이 제한에 걸리는지는 직접 돌려봐야 안다.

**시스템 네임스페이스 로그가 디버깅의 열쇠다.** 유저 네임스페이스의 driver는 `internal error`만 출력한다. 실제 원인은 시스템 네임스페이스의 driver 로그에 있다. LakeFormation 연동 잡의 트러블슈팅에서 이 로그를 보지 않으면 원인 파악이 불가능하다.

**라벨 셀렉터 하나가 전체를 멈출 수 있다.** Spark 앱 이름을 오버라이드하는 것은 관리/모니터링 목적의 일반적인 관행이다. 하지만 EMR이 자동 생성하는 headless service의 라벨 셀렉터가 이 값에 의존하면서, 서비스 디스커버리가 깨졌다. 자동 생성되는 리소스의 라벨 구조까지 확인해야 한다.

**새 기능의 제한사항은 문서보다 실험이 빠르다.** FGAC가 어떤 Spark 기능을 차단하는지는 공식 문서에 전부 나와 있지 않다. 10개의 이슈 대부분은 직접 잡을 돌려보면서 발견했다. PoC 없이 프로덕션에 적용했다면 모든 팀의 파이프라인이 동시에 실패했을 것이다.

**참고 자료:**
- [EMR: Integrate with Apache Ranger](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-ranger.html)
- [EMR: Ranger Architecture](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-ranger-architecture.html)
- [EMR: Ranger Components](https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-ranger-components.html)
- [EMR-on-EKS: FGAC Troubleshooting](https://docs.aws.amazon.com/emr/latest/EMR-on-EKS-DevelopmentGuide/security_iam_fgac-troubleshooting.html)
- [AWS Lake Formation: Getting Started](https://docs.aws.amazon.com/lake-formation/latest/dg/getting-started-setup.html)
