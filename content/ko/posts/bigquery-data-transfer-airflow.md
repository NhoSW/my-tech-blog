---
title: "BigQuery Data Transfer와 Airflow 통합: 매 배치마다 트랜스퍼를 생성하고 삭제하는 이유"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [bigquery, airflow, data-transfer, gcp, aws, s3]
showTableOfContents: true
summary: "S3 마트 테이블을 BigQuery로 인입하는 파이프라인을 구축했다. PoC에서는 BigQuery Data Transfer 스케줄링을 GCP 쪽에 맡겼지만, 운영으로 가면서 Airflow에 통합했다. 매 배치 틱마다 트랜스퍼 객체를 생성하고 데이터 로드 완료 후 삭제하는 구조다. 사용자 피드백으로 멀티데이 lookback, 동시 실행 쿼터 제한, 빈 소스 경로 감지까지 개선한 과정을 정리한다."
---

S3에 적재된 Gold 테이블을 BigQuery로 인입하는 파이프라인이 필요했다. AWS의 데이터 레이크에서 가공된 마트 테이블을 GCP BigQuery에서 분석할 수 있도록 하는 것이 목표다.

BigQuery Data Transfer Service(DTS)가 이 용도에 맞다. S3에서 BigQuery로 데이터를 직접 로드할 수 있고, 별도 컴퓨팅 비용이 없다. 문제는 이 DTS의 스케줄링을 어디서 관리할 것인가였다.

---

## PoC에서의 접근과 한계

PoC 단계에서는 빠르게 동작을 확인하는 것이 목적이었다. DTS 트랜스퍼 객체를 1회성 Airflow 잡으로 생성하고, 실제 스케줄링은 BigQuery 서비스 자체에 맡겼다. BigQuery 콘솔에서 트랜스퍼 스케줄을 확인하고 관리하는 방식이다.

동작은 했지만 운영 관점에서 문제가 있었다. 트랜스퍼 스케줄이 Airflow에 통합되어 있지 않다. Airflow에서는 트랜스퍼가 완료됐는지 알 수 없고, 후속 태스크와의 의존성도 관리할 수 없다. 트랜스퍼 실패 시 알림도 BigQuery 쪽에서만 확인할 수 있다.

대안으로 BigQuery 테이블 센서를 사용하는 방법도 있었다. 트랜스퍼가 완료되면 테이블에 데이터가 쌓이니, 센서로 테이블 업데이트를 감지하는 방식이다. 하지만 이건 간접적인 확인이다. 트랜스퍼가 실패한 건지, 아직 진행 중인 건지, 소스에 데이터가 없는 건지 구분이 안 된다.

매 배치 틱마다 트랜스퍼 객체의 생성과 삭제를 모두 Airflow에서 관리하는 것이 구조적으로 올바르다고 판단했다.

---

## 설계: 매 배치마다 생성하고 삭제한다

핵심 아이디어는 트랜스퍼 객체를 상시 유지하지 않는 것이다.

1. Airflow DAG의 배치 틱이 시작되면 DTS 트랜스퍼 객체를 생성한다
2. 트랜스퍼가 완료될 때까지 대기한다 (async 오퍼레이터 사용)
3. 완료되면 트랜스퍼 객체를 삭제한다

이 방식의 장점은 명확하다. Airflow가 트랜스퍼의 전체 라이프사이클을 관리하므로 실패 감지, 재시도, 후속 태스크 의존성이 모두 Airflow의 기존 메커니즘으로 처리된다. BigQuery 콘솔에 유령 트랜스퍼 객체가 쌓이지도 않는다.

기본 쓰기 모드는 `WRITE_TRUNCATE`다. 각 시간 파티션별로 적재 시 기존 데이터를 덮어쓰므로 멱등성이 보장된다. 같은 배치를 여러 번 실행해도 결과가 동일하다.

### 구현

Apache Airflow의 Google Cloud Provider에 BigQuery DTS 오퍼레이터가 있지만, 우리 유스케이스에 맞지 않는 부분과 미수정 버그가 있었다. 몇 가지 커스터마이징과 버그 픽스를 진행하고 내부 프로바이더 패키지(`airfilter-providers`)에 포함했다. 수정한 버그는 Airflow 원본 레포에도 MR을 올려두었다.

async 오퍼레이터를 활용하기 위해 Airflow triggerer 컴포넌트에 관련 볼륨 마운트도 추가했다. triggerer는 deferrable 오퍼레이터가 외부 이벤트를 기다릴 때 사용하는 Airflow 컴포넌트로, DTS 트랜스퍼 완료를 비동기로 대기하는 데 필요하다.

### 사용자 인터페이스

실제 사용자가 해야 할 일은 설정 파일에 `S3ToBigQueryLoadConfig` 객체를 추가하는 것뿐이다.

```python
S3_LOAD_CONFIG_BY_DAG_ID = S3ToBigQueryLoadConfigByDagIdDict(
    dataeng_bigquery_dts_load_dag_v1=[
        S3ToBigQueryLoadConfig(
            src_s3_path='s3://bucket/schema/table_name',
            daily_partition_field='part_dt',
            require_partition_filter=True,
            time_partition_expiration_days=365,
        ),
    ],
)
```

S3 소스 경로, 파티션 필드, 만료 기간만 지정하면 나머지는 프레임워크가 처리한다. 싱크 데이터셋과 테이블명은 기본값으로 S3 경로의 마지막 두 요소를 사용하고, 필요하면 `sink_dataset`과 `sink_table` 인자로 직접 지정할 수 있다.

---

## Spark 직접 인제션 대비 DTS의 장점

S3 데이터를 BigQuery에 넣는 다른 방법도 있다. EMR Spark에서 데이터프레임을 만들어 BigQuery에 직접 쓰는 방식이다. 이 방식 대비 DTS를 선택한 이유가 있다.

**비용:** DTS를 통한 데이터 로드에는 별도 컴퓨팅 비용이 없다. AWS에서 GCP로의 네트워크 비용은 어떤 방식이든 동일하게 발생하지만, DTS는 로드 자체의 컴퓨팅이 무료다. Spark 잡을 돌리면 EMR 클러스터 비용이 추가된다.

**유지보수:** Spark 인제션은 Spark 앱 생성, 파티션 필드 제거 로직, BigQuery 커넥터 설정 등을 관리해야 한다. DTS 래퍼는 설정 객체 하나 추가하면 끝이다. 향후 config.yaml로 설정을 받는 형태로 전환하면 더 간소화할 수 있다.

---

## 사용자 피드백과 개선

실제 사용자가 23개 테이블을 이관하면서 네 가지 피드백을 공유해주셨다. 실 사용에서 나온 피드백이라 모두 타당했다.

### 멀티데이 lookback

**문제:** 기존 구현은 전일자(D-1) 고정 적재만 가능했다. 하지만 일부 테이블은 매일 14일치 파티션이 업데이트된다. 14일치 데이터를 모두 갱신해야 하는데, 1일치만 적재할 수 있었다.

이건 CDC 테이블이 아니라 배치 테이블에서 발생하는 패턴이다. 원천 시스템에서 과거 데이터를 소급 수정하는 경우, 최근 N일의 파티션이 매일 갱신된다.

**대응:** `S3ToBigQueryLoadConfig`에 `lookback_window_days` 인자를 추가했다. 기본값은 0(전일자만)이고, 예를 들어 14로 설정하면 D-1부터 D-14까지 14개 파티션을 매 배치마다 적재한다.

### 동시 실행 쿼터 제한

**문제:** 20개 이상의 테이블 로드 태스크가 동시에 실행되면, BigQuery 쪽의 throughput 쿼터 제한으로 일부 태스크가 강제 실패했다.

BigQuery에는 프로젝트 레벨의 동시 로드 잡 제한이 있다. 한꺼번에 너무 많은 DTS 트랜스퍼가 실행되면 쿼터를 초과한다.

**대응:** BigQuery 쪽에 쿼터 증설을 요청할 수도 있지만, Airflow 쪽에서 동시 실행 수를 제한하는 것이 더 안전하다. `bigquery_dts_pool`이라는 별도 슬롯 풀을 구성하고, DTS 트랜스퍼 태스크에 이 풀을 매핑했다. 슬롯 풀의 크기로 동시 실행 수를 제어한다.

### 빈 소스 경로 감지

**문제:** S3 소스 경로에 파일이 없는 경우(사용자 설정 실수 등), DTS 트랜스퍼가 에러 없이 성공으로 처리됐다. BigQuery에는 테이블 이름만 생성되고 데이터는 비어 있는 상태가 된다. 작업자는 성공했다고 생각하지만 실제로는 데이터가 없다.

이건 DTS 자체의 동작이다. 소스 경로에 파일이 없으면 로드할 것이 없으니 "할 일을 다 했다"고 판단하고 성공을 반환한다.

**대응:** 소스 경로에 파일이 없는 것과 잡이 실패하는 것을 구분하기 위해 `AirflowSkipException`을 레이즈하도록 구성했다. 파일이 없으면 태스크가 skip 상태가 되어, 성공과 실패 어느 쪽도 아닌 "건너뜀"으로 표시된다.

여기서 한 가지 어려움이 있었다. DTS의 TransferRun 응답 객체에는 "소스에 파일이 없었다"는 정보가 담겨 있지 않다. 이 정보는 GCP의 로깅 서비스에만 기록된다. Airflow의 Google Cloud Provider에는 GCP 로깅 클라이언트 연동 훅이 제공되지 않아서, 커스텀 훅을 구현해서 GCP 로깅 API로부터 관련 로그를 추출하는 방식으로 처리했다.

### 타임존 처리

DTS의 스케줄링 타임존과 Airflow의 타임존이 다를 수 있다. Airflow가 KST로 동작하는데 DTS가 UTC로 트랜스퍼를 생성하면 날짜가 어긋난다.

이 부분은 초기 구현부터 timezone 보정 인자를 제공하고 있었지만, 사용자에게 가이드가 부족했다. UTC/KST 기준에 대한 안내를 위키 문서에 추가했다.

---

## 파티션 매핑

S3의 Hive 테이블과 BigQuery 테이블의 파티션 구조가 다르다는 점도 처리해야 했다.

BigQuery 테이블은 최대 1개의 파티션 필드만 지원한다. 하지만 S3의 Hive 테이블 중에는 `LOG_DATE`와 `LOG_HOUR` 두 단계로 파티션된 것이 있다. 이런 테이블은 BigQuery의 pseudo column인 `_PARTITIONTIME`으로 매핑했다.

예를 들어 `dt=2022-08-15/hour=3`으로 파티션된 데이터는 `_PARTITIONTIME$2022081503` 파티션에 들어간다. `_PARTITIONTIME`이 datetime 타입이므로 날짜와 시간을 하나의 필드로 합쳐서 표현할 수 있다.

---

## WRITE_APPEND 모드 고려

기본 모드인 `WRITE_TRUNCATE`는 멱등성이 보장되지만, `WRITE_APPEND` 모드를 사용해야 하는 경우도 있다. 이력을 누적해야 하는 테이블이 그렇다.

`WRITE_APPEND`에서는 재실행 시 데이터가 중복된다. 멱등성을 보장하려면 DTS 트랜스퍼 객체를 삭제하지 않고 유지하는 방식으로 수정해야 한다. 같은 트랜스퍼 객체의 같은 실행은 중복 적재되지 않기 때문이다. 이 부분은 별도 검토 대상으로 분리했다.

---

## 배운 것

**스케줄링은 한곳에서 관리해야 한다.** DTS 자체 스케줄링과 Airflow 스케줄링이 분리되면, 실패 감지, 재시도, 의존성 관리가 모두 이중으로 필요해진다. 매 배치마다 트랜스퍼를 생성하고 삭제하는 구조가 복잡해 보이지만, 운영 측면에서는 가장 단순하다.

**DTS의 "성공"을 맹신하지 말자.** 소스에 파일이 없어도 DTS는 성공을 반환한다. "성공 = 데이터가 적재됨"이라는 가정은 위험하다. 실제로 데이터가 있는지 별도 검증이 필요하다.

**쿼터 제한은 클라이언트 쪽에서 제어하는 게 낫다.** BigQuery 쪽에 쿼터 증설을 요청할 수도 있지만, 증설 후에도 더 많은 테이블이 추가되면 다시 한계에 부딪힌다. Airflow 슬롯 풀로 동시 실행 수를 직접 제어하면 쿼터 범위 안에서 안정적으로 운영할 수 있다.

**사용자 피드백은 설계를 완성한다.** 초기 구현은 "D-1 파티션 1개 적재"만 고려했다. 실제 사용자가 23개 테이블을 이관하면서 멀티데이 lookback, 동시 실행 제한, 빈 경로 감지 등 운영에서만 드러나는 요구사항이 나왔다. 프레임워크는 실사용 피드백 없이 완성될 수 없다.

**참고 자료:**
- [BigQuery Data Transfer Service Pricing](https://cloud.google.com/bigquery/pricing#bqdts)
- [BigQuery: Introduction to Loading Data](https://cloud.google.com/bigquery/docs/loading-data)
- [BigQuery: Batch Loading Data](https://cloud.google.com/bigquery/docs/batch-loading-data)
- [BigQuery Quotas: Load Jobs](https://cloud.google.com/bigquery/quotas#load_jobs)
- [Airflow: Google Cloud BigQuery DTS Operators](https://airflow.apache.org/docs/apache-airflow-providers-google/stable/operators/cloud/bigquery_dts.html)
