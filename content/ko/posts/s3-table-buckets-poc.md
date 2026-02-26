---
title: "S3 테이블 버킷 도입 검토: 매니지드 Iceberg의 가능성과 한계"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [iceberg, s3, aws, trino, spark, kafka-connect, compaction, managed-service]
showTableOfContents: true
summary: "AWS S3 테이블 버킷은 자동 컴팩션을 제공하는 매니지드 Iceberg다. CDC 싱크 테이블의 컴팩션 문제를 해결할 수 있을지 PoC를 진행했다. Trino, Spark, Kafka Connect 연동을 확인하고, 자동 컴팩션의 동작 특성과 비용을 검토했다. 결론은 모든 테이블에 쓸 서비스는 아니고, CDC 테이블에 한해서 가치가 있다는 것이다."
---

2025년 3월에 S3 테이블 버킷이 서울 리전에 출시됐다. 한 줄로 요약하면 매니지드 Iceberg 테이블이다. AWS가 테이블의 자동 컴팩션과 스냅샷 관리를 제공하고, 일반 S3 버킷보다 높은 TPS와 스루풋을 제공한다고 한다.

이 서비스가 눈에 들어온 이유는 명확하다. CDC 싱크 테이블의 컴팩션이 골칫거리였다.

CDC 테이블의 컴팩션 문제는 이렇다. 시간 파티션이 걸려 있어도 원천 테이블이 히스토리성이 아니면 전 기간 파티션에 걸쳐 update와 delete가 발생한다. 예를 들어 사용자 프로필 테이블 같은 경우, 3년 전에 가입한 사용자가 오늘 닉네임을 바꾸면 3년 전 파티션에 delete 마커와 새 레코드가 쌓인다. 컴팩션 대상 시간 범위를 최근 5일로 설정하면 그 밖의 파티션은 정리되지 않고, 30일로 늘려도 마찬가지다. 전 기간 컴팩션은 테이블 크기에 따라 비용이 수십 달러에서 수백 달러까지 들 수 있고, 실행 시간도 길어서 다른 쿼리에 영향을 준다.

자동 컴팩션이 이 문제를 해결해줄 수 있을까? PoC를 진행했다.

---

## 매니지드 서비스에 대한 우려

기대만큼 우려도 있었다. 세 가지가 특히 마음에 걸렸다.

첫째, 블랙박스 문제다. 매니지드 서비스는 이슈가 발생하면 내부를 들여다볼 수 없다. 컴팩션이 동작하지 않을 때 왜 안 되는지를 확인하는 수단이 한정적이다. 게다가 출시된 지 몇 달밖에 안 된 서비스라 실사용 사례나 트러블슈팅 레퍼런스가 거의 없었다.

둘째, 과거의 쓰라린 경험이다. 사실 S3 테이블 버킷 이전에 AWS가 Iceberg 자동 컴팩션을 제공한 적이 있다. Glue/LakeFormation의 하위 기능으로 출시된 매니지드 컴팩션이었는데, 내부적으로 Glue 기반 Spark 프로시저 호출 잡으로 수행됐다. 문제는 대용량 테이블에서 별다른 에러 로그도 없이 조용히 실패했다는 것이다. 수백 개의 파일이 쌓여있는데 컴팩션은 일어나지 않고, Glue 잡 히스토리에도 성공으로 찍혀 있었다. 결국 자체 컴팩션 배치로 돌아갔다. 같은 일이 반복될 수 있다는 경계심이 있었다.

셋째, 비용이다. S3 테이블 버킷의 스토리지 비용이 일반 S3 버킷보다 높다. 여기에 자동 컴팩션 처리 비용도 추가된다. 매니지드의 편의성이 추가 비용을 상쇄하는지 따져봐야 했다.

---

## Glue REST Catalog 연동

S3 테이블 버킷은 Glue Data Catalog의 Iceberg REST 엔드포인트를 통해 접근한다. 기존에 사용하던 Glue 타입 카탈로그(`type=glue`)가 아닌 REST 타입 카탈로그(`type=rest`)를 별도로 구성해야 한다.

이 아키텍처 결정은 후속 제약사항에 연쇄적으로 영향을 미쳤다.

### 기존 테이블과 동시 관리 불가

PoC에서 가장 먼저 확인한 사실이다. 단일 Iceberg 카탈로그로 S3 테이블 버킷의 테이블과 기존 일반 S3 버킷의 Iceberg 테이블을 동시에 관리할 수 없다. 근본적인 이유는 LakeFormation 권한 정책 자체가 카탈로그 레벨에서 분리되어 있기 때문이다. REST 타입 카탈로그와 Glue 타입 카탈로그는 LakeFormation에서 서로 다른 권한 체계로 관리된다.

이는 사용자 경험에 직접 영향을 준다. Trino에서 별도 카탈로그를 추가해야 하고, 사용자는 어떤 테이블이 어느 카탈로그에 있는지 알아야 한다.

```
기존 테이블:      iceberg.raw_log.weblog_common
테이블 버킷 테이블: iceberg_managed.raw_log.weblog_common_managed
```

쿼리에서 기존 테이블과 테이블 버킷 테이블을 조인하려면 크로스 카탈로그 쿼리를 작성해야 한다. 자주 일어나는 패턴은 아니지만, 마이그레이션 검증 과정에서는 두 테이블의 데이터를 비교해야 하는 경우가 있어서 번거로웠다.

### DDL 제한

REST 타입 카탈로그에서는 일부 DDL이 지원되지 않는다. 이건 Trino의 제약이 아니라 Glue REST API 자체의 제약이다.

- **CREATE TABLE**: Glue REST Catalog가 Trino Iceberg 커넥터가 의존하는 stage-create API를 지원하지 않는다. 테이블 생성은 AWS 콘솔이나 Spark에서 해야 한다.
- **RENAME TABLE**: Glue REST API에서 rename 연산을 지원하지 않는다고 공식 문서에 명시되어 있다.

SELECT, INSERT 등 읽기/쓰기 작업은 정상 동작한다. Hive → Iceberg 테이블 리디렉션도 동작하지만, Iceberg → Hive 리디렉션은 REST 카탈로그에서 지원되지 않는다.

CREATE TABLE이 안 되는 건 운영 워크플로우에서 꽤 불편하다. 새 CDC 테이블을 추가할 때마다 Spark 세션을 열어서 테이블을 생성해야 한다. 다만 CDC 싱크용 테이블은 Kafka Connect의 Iceberg 커넥터가 자동 생성해주기 때문에, 실제로는 스키마와 테이블 버킷 네임스페이스만 미리 만들어두면 된다.

### LakeFormation 권한

S3 테이블 버킷은 기본적으로 LakeFormation 연동이 활성화되어 있다. Trino 워커, Kafka Connect, Spark 등 데이터에 접근하는 모든 IAM role에 대해 LakeFormation에서 테이블, 스키마, 카탈로그 단위 권한을 명시적으로 부여해야 한다.

여기서 가장 까다로운 점은 권한이 없을 때의 동작이다. 에러가 발생하는 게 아니라, 해당 카탈로그에 아무 테이블도 없는 것처럼 보인다. `SHOW TABLES`를 실행하면 빈 결과가 반환되고, `SELECT`를 하면 "Table not found" 에러가 나온다. 테이블이 정말 없는 건지 권한이 없는 건지 구분이 안 된다.

PoC 초반에 이걸로 시간을 꽤 잃었다. Trino 카탈로그 설정을 세 번이나 바꿔봤는데, 실은 LakeFormation 권한이 누락된 거였다. 이 경험 이후에 S3 테이블 버킷 관련 문제가 발생하면 가장 먼저 LakeFormation 권한부터 확인하는 체크리스트를 만들었다.

---

## Trino 연동

Trino v471부터 S3 테이블 버킷에 대한 읽기가 지원된다. 현재 운영 중인 v476에서 읽기와 쓰기 모두 사용 가능하다.

### Fault-Tolerant 모드 비호환

PoC 중 확인한 중요한 제약이다. Trino의 fault-tolerance 설정(`retry_policy`)이 활성화된 상태에서 S3 테이블 버킷에 쓰기를 시도하면 에러가 발생한다.

원인으로 추정되는 것은 S3 테이블 버킷의 내부 경로 구조다. 테이블 버킷은 일반 S3와 경로 구조가 다르다. Trino의 fault-tolerant 모드는 쓰기 중간 결과를 리스팅해서 재시도 여부를 판단하는데, 이 리스팅 API가 테이블 버킷의 경로 구조와 호환되지 않는 것으로 보인다. Trino 커뮤니티에 이슈가 열려 있지만 아직 해결되지 않았다.

우회 방법은 해당 쿼리 세션에서 `retry_policy`를 `NONE`으로 지정하는 것이다. fault-tolerant 모드를 끄면 정상적으로 쓰기가 동작한다. 다만 이 설정은 세션 레벨이기 때문에, 사용자에게 S3 테이블 버킷에 쓸 때는 이 설정을 명시하라고 안내해야 한다. 배치 잡에서는 코드에 미리 넣어두면 되지만, 인터랙티브 쿼리에서는 매번 잊지 않고 설정해야 하는 번거로움이 있다.

---

## Spark 연동

EMR Spark에서도 Glue REST Catalog 엔드포인트를 통해 S3 테이블 버킷의 Iceberg 테이블을 정상 조회할 수 있다.

연동 방법이 두 가지다. AWS가 오픈소스로 제공하는 S3 Tables Catalog 라이브러리를 사용하는 방법과, Glue REST 엔드포인트를 직접 활용하는 방법이다. S3 Tables Catalog 라이브러리는 EMR v7.5 이상이 필요한데, 우리 환경에서는 아직 EMR v6.14를 사용하고 있어서 Glue REST 엔드포인트를 직접 사용했다. EMR v6.14에서도 문제 없이 동작한다.

Spark 세션 설정:

```properties
spark.sql.catalog.iceberg_managed=org.apache.iceberg.spark.SparkCatalog
spark.sql.catalog.iceberg_managed.type=rest
spark.sql.catalog.iceberg_managed.uri=https://glue.ap-northeast-2.amazonaws.com/iceberg
spark.sql.catalog.iceberg_managed.rest.sigv4-enabled=true
spark.sql.catalog.iceberg_managed.rest.signing-name=glue
```

Trino와 마찬가지로 LakeFormation에서 EMR Spark용 IAM role에 권한을 부여해야 한다. 권한이 없을 때의 동작도 동일하다. 에러 없이 빈 결과가 반환된다. Spark에서 `SHOW TABLES`를 실행했는데 아무것도 안 나오면, 십중팔구 LakeFormation 권한 문제다.

---

## Kafka Connect 연동

가장 중요한 연동 검증이었다. CDC 싱크의 메인 파이프라인이 Kafka Connect를 통해 동작하기 때문이다.

Kafka Connect의 Iceberg 싱크 커넥터에서 Glue REST 엔드포인트를 통해 S3 테이블 버킷으로의 실시간 싱크가 정상 동작함을 확인했다.

CDC 토픽의 경우 Debezium 소스 커넥터가 프로듀스하는 메시지에 스키마 정보가 포함되어 있어서 테이블 자동 생성과 스키마 에볼루션이 정상 동작한다. 원천 테이블에 컬럼이 추가되면 싱크 측 Iceberg 테이블에도 자동으로 컬럼이 추가되는 것까지 확인했다.

다만 스키마 없이 인입되는 웹 로그 같은 데이터의 경우에는 타임스탬프 타입 추론이 불가능해서 테이블을 미리 생성해둬야 한다. 타임스탬프 필드가 단순 문자열로 들어오면 Iceberg 커넥터가 이를 `string` 타입으로 생성하는데, 나중에 `timestamp` 타입으로 변경하려면 테이블을 다시 만들어야 한다. 이 문제는 S3 테이블 버킷 특유의 이슈가 아니라 Iceberg 싱크 커넥터 자체의 한계인데, 테이블 버킷에서는 CREATE TABLE이 Trino에서 안 되니까 테이블 재생성이 더 번거로워진다.

---

## 자동 컴팩션 검토

### 동작 확인

S3 테이블 버킷에 생성된 모든 테이블에는 완전관리형 컴팩션이 자동 활성화된다. 기본 타겟 파일 크기는 512MB이고 64MB에서 512MB 사이에서 조정 가능하다.

PoC에서 가장 알고 싶었던 건 자동 컴팩션의 트리거 조건이다. 결론부터 말하면, 트리거 조건은 문서화되어 있지 않다. AWS CLI를 통해 잡 수행 히스토리는 확인할 수 있지만, "어떤 조건일 때 컴팩션이 시작되는가"에 대한 정보는 공개되지 않았다. 구체적인 조건을 알려면 서포트 케이스를 열어서 문의해야 한다.

실제 관찰 결과, 현재 1시간 단위 배치로 수행하는 수동 컴팩션에 비해 트리거 조건이 상당히 보수적으로 보였다. 수동 컴팩션에서는 파일이 일정 개수 이상 쌓이면 바로 실행하는데, 자동 컴팩션은 파일이 꽤 많이 쌓인 후에야 실행됐다. 빈도 자체가 낮다는 얘기다.

이건 관점에 따라 장단이 있다. 보수적으로 동작하면 불필요한 컴팩션 비용이 줄지만, 그 사이에 쌓인 작은 파일들이 쿼리 성능에 영향을 줄 수 있다. CDC 테이블처럼 작은 파일이 지속적으로 쌓이는 워크로드에서는 좀 더 공격적인 컴팩션이 필요할 수도 있다.

### 비용

스토리지 비용과 읽기/쓰기 비용 외에 자동 컴팩션에 따른 비용이 추가된다. 과금 기준은 두 가지다.

- 처리된 객체 1,000개당 $0.004
- 처리된 GB당 $0.05

구체적인 예시로 계산해보면, 3만 개의 5MB 파일(약 146GB)을 컴팩션하는 경우:
- 객체 처리 비용: 30,000 / 1,000 × $0.004 = $0.12
- 데이터 처리 비용: 146 × $0.05 = $7.30
- 합계: 약 $7.44

수동 컴팩션의 경우 EMR Spark 잡이나 Trino 쿼리로 수행하는데, 같은 양을 처리하면 EC2 인스턴스 비용이 이보다 더 들 수 있다. 거기에 배치 잡 관리, 실패 모니터링, 재시도 로직 같은 운영 공수까지 감안하면 자동 컴팩션의 비용은 충분히 받아들일 만하다.

### append-only 테이블에는 불필요

PoC를 진행하면서 명확해진 결론이 하나 있다. 모든 테이블에 S3 테이블 버킷을 쓸 필요는 없다.

append-only로 동작하는 로그 테이블은 시간 파티션별로 데이터가 순차 적재된다. 최근 파티션에만 새 파일이 쌓이고, 과거 파티션은 건드릴 일이 없다. 이런 테이블은 기존 수동 컴팩션으로 충분하다. 1시간 배치로 최근 파티션만 컴팩션하면 되니까 범위도 명확하고 비용도 예측 가능하다.

자동 컴팩션의 보수적인 트리거 조건을 감안하면, append-only 테이블에 대해서는 현재 수동 컴팩션이 비용과 성능 면에서 더 합리적이다.

S3 테이블 버킷이 진짜 가치를 발휘하는 건 CDC 테이블이다. 어떤 파티션에서 update나 delete가 발생할지 예측할 수 없고, 그래서 컴팩션 대상 범위를 특정하기 어려운 워크로드. 이런 테이블에 한해서 자동 컴팩션의 편의성이 비용을 상쇄한다.

---

## CDC 싱크 테이블 PoC

append-only 로그 테이블에 대한 PoC에 이어서, 실제로 업데이트가 지속적으로 발생하는 CDC 테이블에 대해 추가 PoC를 진행했다. 두 가지를 확인하려 했다.

첫째, 자동 컴팩션이 CDC 싱크 커밋과 충돌 없이 동작하는가. Kafka Connect가 데이터를 커밋하는 중에 자동 컴팩션이 동시에 실행되면 충돌이 발생할 수 있다. Iceberg의 낙관적 동시성 제어가 이를 처리하지만, 실제로 잘 동작하는지는 확인이 필요했다.

둘째, 기존 수동 컴팩션 대비 유지보수성이 실제로 개선되는가. 비용이 비슷하더라도 운영자가 컴팩션 배치를 관리할 필요가 없어지면 그 자체로 가치가 있다.

운영 도입 계획은 이렇다. 신규 CDC 테이블 싱크 요청이 들어올 때 기존 S3 버킷과 S3 테이블 버킷에 2벌의 싱크 파이프라인을 운영한다. 동일한 Kafka 토픽을 두 곳에 동시에 싱크하면서 데이터 정합성과 자동 컴팩션 동작을 검증한다. 충분한 기간 동안 안정성이 확인되면 기존 S3 버킷 쪽 파이프라인을 제거한다.

2벌 운영은 비용이 두 배 드는 비효율처럼 보이지만, 과거 매니지드 컴팩션의 무성의한 실패를 경험한 이후로는 이 정도 검증 비용은 보험이라고 생각한다.

---

## 배운 것

**매니지드 서비스의 자동화가 모든 워크로드에 맞는 건 아니다.** append-only 테이블은 기존 수동 컴팩션이 오히려 더 효율적이다. 자동 컴팩션의 보수적인 트리거 조건과 추가 비용을 감안하면, 수동 컴팩션의 범위가 명확한 워크로드에서는 기존 방식이 합리적이다. 자동 컴팩션은 예측 불가능한 파티션에 update/delete가 발생하는 CDC 테이블에 한해서 가치가 있다.

**카탈로그 분리는 피할 수 없다.** S3 테이블 버킷용 REST 카탈로그와 기존 Glue 카탈로그를 단일 Trino 카탈로그로 통합할 수 없다. LakeFormation 권한 체계가 카탈로그 레벨에서 분리되어 있기 때문이다. 사용자에게 별도 카탈로그명을 안내하고, 필요하면 크로스 카탈로그 쿼리를 사용해야 한다.

**LakeFormation 권한 누락은 에러가 아닌 빈 결과로 나타난다.** 이건 S3 테이블 버킷을 쓰면서 가장 시간을 많이 잃은 부분이다. "테이블이 없습니다"가 아니라 "권한이 없습니다"라고 에러를 던져야 하는데, 그렇지 않다. 새로운 IAM role이 S3 테이블 버킷에 접근할 때는 반드시 LakeFormation 권한을 먼저 확인하는 체크리스트가 필요하다.

**과거 매니지드 컴팩션의 실패 경험을 잊지 말자.** Glue/LakeFormation의 매니지드 컴팩션은 조용히 실패했다. S3 테이블 버킷이 같은 전철을 밟지 않는다는 보장은 없다. 충분한 PoC와 2벌 파이프라인 검증이 그래서 필요하다. 신뢰는 관찰된 안정성으로부터 오는 것이지, 서비스 소개 문서로부터 오는 게 아니다.

**참고 자료:**
- [Amazon S3 Tables](https://aws.amazon.com/s3/features/tables/)
- [New Amazon S3 Tables: Storage optimized for analytics workloads](https://aws.amazon.com/blogs/aws/new-amazon-s3-tables-storage-optimized-for-analytics-workloads/)
- [How Amazon S3 Tables use compaction to improve query performance by up to 3x](https://aws.amazon.com/blogs/storage/how-amazon-s3-tables-use-compaction-to-improve-query-performance-by-up-to-3-times/)
- [Working with Amazon S3 Tables](https://docs.aws.amazon.com/AmazonS3/latest/userguide/s3-tables.html)
- [Trino: Add read support for S3 Tables in Iceberg (v471)](https://github.com/trinodb/trino/pull/24815)
- [Trino: Fault tolerant mode fails with S3 Table Buckets](https://github.com/trinodb/trino/issues/25481)
- [Iceberg Connector - Trino Documentation](https://trino.io/docs/current/connector/iceberg.html)
- [AWS S3 Tables Catalog](https://github.com/awslabs/s3-tables-catalog)
