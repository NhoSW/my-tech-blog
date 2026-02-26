---
title: "S3 테이블 버킷 도입 검토: 매니지드 Iceberg의 가능성과 한계"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [iceberg, s3, aws, trino, spark, kafka-connect, compaction, managed-service]
showTableOfContents: true
summary: "AWS S3 테이블 버킷은 자동 컴팩션을 제공하는 매니지드 Iceberg다. CDC 싱크 테이블의 컴팩션 문제를 해결할 수 있을지 PoC를 진행했다. Trino, Spark, Kafka Connect 연동을 확인하고 자동 컴팩션 동작과 비용을 검토했다."
---

2025년 3월에 S3 테이블 버킷이 서울 리전에 출시됐다. 한 줄로 요약하면 매니지드 Iceberg 테이블이다. 자동 컴팩션과 스냅샷 관리를 제공하고 일반 S3 버킷보다 높은 TPS와 스루풋을 제공한다고 한다.

현재 CDC 싱크 테이블 운영에서 컴팩션이 골칫거리다. 시간 파티션이 걸려 있어도 원천 테이블이 히스토리성이 아니면 전 기간 파티션에 걸쳐 update/delete가 발생한다. 컴팩션 대상 시간 범위를 최근 5일이나 30일로 설정해도 그 밖의 파티션에 쌓이는 파일은 정리되지 않는다. 전 기간 컴팩션은 비용과 관리 면에서 현실적이지 않다.

자동 컴팩션이 이 문제를 해결해줄 수 있을까? PoC를 진행했다.

---

## 매니지드 서비스에 대한 우려

기대도 있었지만 우려도 있었다.

첫째, 블랙박스 문제다. 매니지드 서비스에서 이슈가 발생하면 내부를 들여다볼 수 없다. 출시된 지 얼마 안 된 서비스라 실사용 사례도 적다.

둘째, 과거 경험이다. Glue/LakeFormation의 하위 기능으로 출시됐던 Iceberg 매니지드 컴팩션 기능을 써본 적이 있다. 내부적으로 Glue 기반 Spark 프로시저 호출 잡으로 수행되는데, 대용량 테이블에서 별다른 에러 로그도 없이 실패했다.

셋째, 비용이다. 스토리지 비용이 일반 버킷보다 높고 자동 컴팩션에 따른 추가 비용도 발생한다.

---

## Glue REST Catalog 연동

S3 테이블 버킷은 Glue Data Catalog의 Iceberg REST 엔드포인트를 통해 접근한다. 기존 Glue 카탈로그(glue 타입)가 아닌 REST 타입 카탈로그를 사용해야 한다.

### 기존 테이블과 동시 관리 불가

가장 먼저 확인한 사실이다. 단일 Iceberg 카탈로그로 S3 테이블 버킷의 테이블과 기존 일반 S3 버킷의 Iceberg 테이블을 동시에 관리할 수 없다. LakeFormation 권한 정책 자체가 카탈로그 레벨에서 분리되어 있다.

별도 Trino 카탈로그를 추가해야 한다.

```
기존 테이블:      iceberg.raw_log.weblog_common
테이블 버킷 테이블: iceberg_managed.raw_log.weblog_common_managed
```

### DDL 제한

REST 타입 카탈로그에서는 일부 DDL이 지원되지 않는다.

- **CREATE TABLE**: Glue REST Catalog가 Trino Iceberg 커넥터가 의존하는 stage-create API를 지원하지 않는다
- **RENAME TABLE**: Glue REST API에서 지원하지 않는다고 공식 문서에 명시되어 있다

SELECT, INSERT 등 읽기/쓰기 작업은 정상 동작한다. Hive → Iceberg 테이블 리디렉션은 되지만, Iceberg → Hive 리디렉션은 REST 카탈로그에서 지원되지 않는다.

### LakeFormation 권한

S3 테이블 버킷은 기본적으로 LakeFormation 연동이 활성화되어 있다. Trino 워커, Kafka Connect, Spark 등 데이터에 접근하는 모든 IAM role에 대해 LakeFormation에서 테이블/스키마/카탈로그 단위 권한을 명시적으로 부여해야 한다.

권한이 없으면 에러가 발생하는 게 아니라, 해당 카탈로그에 아무 테이블도 없는 것처럼 보인다. 디버깅하기 까다로운 부분이다.

---

## Trino 연동

Trino v471부터 S3 테이블 버킷에 대한 읽기가 지원된다. 현재 운영 중인 v476에서 사용 가능하다.

### Fault-Tolerant 모드 비호환

Trino의 fault-tolerance 설정(`retry_policy`)이 활성화된 상태에서 S3 테이블 버킷에 쓰기를 시도하면 에러가 발생한다. S3 테이블 버킷의 내부 경로 구조가 일반 S3와 달라서 파일 리스팅 API가 호환되지 않는 것으로 보인다. Trino 커뮤니티에 이슈가 열려 있다.

우회 방법은 해당 쿼리 세션에서 `retry_policy`를 `NONE`으로 지정하는 것이다. 이렇게 하면 정상적으로 쓰기가 동작한다.

---

## Spark 연동

EMR Spark에서도 Glue REST Catalog 엔드포인트를 통해 S3 테이블 버킷의 Iceberg 테이블을 정상 조회할 수 있다.

AWS가 오픈소스로 제공하는 S3 Tables Catalog 라이브러리를 사용하려면 EMR v7.5 이상이 필요하지만, Glue REST 엔드포인트를 직접 활용하면 EMR v6.14에서도 연동 가능하다.

Spark 세션 설정 예시:

```properties
spark.sql.catalog.iceberg_managed=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.iceberg_managed.type=rest
spark.sql.catalog.iceberg_managed.uri=https://glue.ap-northeast-2.amazonaws.com/iceberg
spark.sql.catalog.iceberg_managed.rest.sigv4-enabled=true
spark.sql.catalog.iceberg_managed.rest.signing-name=glue
```

Trino와 마찬가지로 LakeFormation에서 EMR Spark용 IAM role에 권한을 부여해야 한다. 권한이 없으면 에러 없이 빈 결과가 반환된다.

---

## Kafka Connect 연동

Kafka Connect의 Iceberg 싱크 커넥터에서 Glue REST 엔드포인트를 통해 S3 테이블 버킷으로의 실시간 싱크를 확인했다.

CDC 토픽의 경우 Debezium 소스 커넥터가 프로듀스하는 메시지에 스키마 정보가 포함되어 있어서 테이블 자동 생성과 스키마 에볼루션이 정상 동작한다. 스키마 없이 인입되는 웹 로그 같은 경우에는 타임스탬프 타입 추론이 불가능해서 테이블을 미리 생성해둬야 한다.

---

## 자동 컴팩션 검토

### 동작 확인

S3 테이블 버킷에 생성된 모든 테이블에는 완전관리형 컴팩션이 자동 활성화된다. 기본 타겟 파일 크기는 512MB이고 64MB~512MB 사이에서 조정 가능하다.

자동 컴팩션이 트리거되는 조건은 문서화되어 있지 않다. AWS CLI를 통해 잡 수행 히스토리는 확인할 수 있지만 트리거 조건 자체는 공개되지 않았다. 필요하면 서포트 케이스로 문의해야 한다.

현재 1시간 단위 배치로 수행하는 수동 컴팩션에 비해 트리거 조건이 상당히 보수적으로 보였다.

### 비용

스토리지 비용과 읽기/쓰기 비용 외에 자동 컴팩션에 따른 비용이 추가된다. 처리된 객체 1,000개당 $0.004, 처리된 GB당 $0.05가 과금된다.

예를 들어 3만 개의 5MB 파일(약 146GB)을 컴팩션하면 약 $7.44가 든다. 수동 컴팩션의 컴퓨팅 비용과 관리 공수를 생각하면 받아들일 만한 수준이다.

### append-only 테이블에는 불필요

append-only로 동작하는 로그 테이블은 시간 파티션별로 데이터가 순차 적재되므로 기존 수동 컴팩션으로 충분하다. 자동 컴팩션의 보수적인 트리거 조건을 감안하면 현재 운영 방식을 유지하는 게 비용과 성능 면에서 합리적이다.

S3 테이블 버킷은 어떤 파티션에서 update/delete가 발생할지 예측하기 어려운 CDC 테이블에 한해 적용하는 것이 합리적이다.

---

## CDC 싱크 테이블 PoC

append-only 로그 테이블에 대한 PoC에 이어서, 실제로 업데이트가 지속적으로 발생하는 CDC 테이블에 대해 추가 PoC를 진행했다. 자동 컴팩션이 CDC 싱크 커밋과 충돌 없이 동작하는지, 기존 수동 컴팩션 대비 유지보수성이 개선되는지 확인하는 것이 목적이다.

운영 계획은 신규 CDC 테이블 싱크 요청이 들어올 때 기존 S3 버킷과 S3 테이블 버킷에 2벌의 싱크 파이프라인을 운영하면서 안정성을 검증하는 방식이다. 검증이 끝나면 기존 S3 버킷 쪽을 제거한다.

---

## 배운 것

**매니지드 서비스의 자동화가 모든 워크로드에 맞는 건 아니다.** append-only 테이블은 기존 수동 컴팩션이 더 효율적이다. 자동 컴팩션은 예측 불가능한 파티션에 update/delete가 발생하는 CDC 테이블에 가치가 있다.

**카탈로그 분리는 피할 수 없다.** S3 테이블 버킷용 카탈로그와 기존 Glue 카탈로그를 단일 Trino 카탈로그로 통합할 수 없다. 사용자에게 별도 카탈로그명을 안내해야 한다.

**LakeFormation 권한 누락은 에러가 아닌 빈 결과로 나타난다.** 디버깅이 어렵다. IAM role에 권한을 미리 부여하는 체크리스트가 필요하다.

**과거 매니지드 컴팩션의 실패 경험을 잊지 말자.** Glue/LakeFormation 매니지드 컴팩션은 조용히 실패했다. S3 테이블 버킷이 같은 전철을 밟지 않는다는 보장은 없다. 충분한 PoC와 2벌 운영으로 검증하는 게 맞다.

**참고 자료:**
- [Amazon S3 Tables](https://aws.amazon.com/s3/features/tables/)
- [New Amazon S3 Tables: Storage optimized for analytics workloads](https://aws.amazon.com/blogs/aws/new-amazon-s3-tables-storage-optimized-for-analytics-workloads/)
- [How Amazon S3 Tables use compaction to improve query performance by up to 3x](https://aws.amazon.com/blogs/storage/how-amazon-s3-tables-use-compaction-to-improve-query-performance-by-up-to-3-times/)
- [Working with Amazon S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [Trino: Add read support for S3 Tables in Iceberg (v471)](https://github.com/trinodb/trino/pull/24815)
- [Trino: Fault tolerant mode fails with S3 Table Buckets](https://github.com/trinodb/trino/issues/25481)
- [Iceberg Connector - Trino Documentation](https://trino.io/docs/current/connector/iceberg.html)
- [AWS S3 Tables Catalog](https://github.com/awslabs/s3-tables-catalog)
