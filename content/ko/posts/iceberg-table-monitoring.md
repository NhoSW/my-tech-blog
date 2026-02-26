---
title: "Iceberg 테이블 모니터링 구축: Trino 메타테이블과 Prometheus Pushgateway"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [iceberg, trino, monitoring, grafana, prometheus, pushgateway, airflow]
showTableOfContents: true
summary: "Iceberg 테이블 수가 늘어나면서 파일 상태를 체계적으로 모니터링할 필요가 생겼다. Trino 메타테이블을 활용해 파일 수, 크기, 파티션 분포를 추적하고 Prometheus Pushgateway와 Grafana로 대시보드를 구성했다. 과정에서 만난 Trino 버그와 성능 이슈도 정리했다."
---

Iceberg 테이블을 운영하다 보면 눈에 보이지 않는 곳에서 문제가 쌓인다. 작은 파일이 계속 늘어나는지, 컴팩션이 제대로 돌고 있는지, 특정 파티션에 데이터가 몰리고 있는지. 테이블 수가 적을 때는 수동으로 확인해도 되지만, 수십 개를 넘어가면 체계적인 모니터링이 필요하다.

카카오의 [로그 유형별 Iceberg 테이블 적재 및 운영 전략](https://tech.kakao.com/posts/694) 글을 보고 본격적으로 모니터링 체계 구축에 착수했다.

---

## 모니터링 대상

Iceberg 메타테이블을 통해 확인해야 할 핵심 항목은 세 가지다.

- **작은 파일 증가 여부**: 작은 파일이 계속 쌓이면 쿼리 성능이 떨어진다. 스캔해야 할 파일 수 자체가 늘어나기 때문이다.
- **최적화 작업 상태**: 컴팩션이 정상적으로 수행되고 있는지. 파일 수와 평균 파일 크기의 추이를 보면 알 수 있다.
- **파티션 분포**: 특정 파티션에 데이터가 몰리는 현상이 있는지. 파티션별 파일 수와 크기를 비교하면 확인할 수 있다.

최신 스냅샷이 참조하는 데이터 파일의 개수, 평균 크기, 파티션별 분포를 주기적으로 수집하면 이 항목들을 모두 커버할 수 있다.

---

## 아키텍처: 두 가지 접근

모니터링 데이터 소스를 어떻게 구성할지 두 가지 방안을 검토했다.

### 1안: Grafana에서 Trino 직접 조회

Grafana에 Trino 데이터소스를 연결하고 대시보드 조회 시마다 메타테이블 쿼리를 직접 제출하는 방식이다.

장점은 구현이 간단하다는 거다. 단점이 많았다.

- 대시보드를 열 때마다 Trino에 쿼리가 제출된다. 거의 같은 결과를 매번 다시 계산하는 셈이다.
- 무거운 테이블의 메타 쿼리는 수 분이 걸린다. 대시보드 로딩이 실용적이지 않다.
- Trino 자체의 제약으로 메트릭 계산 로직을 커스터마이징하기 어렵다.
- Grafana의 Trino 플러그인이 ROW 타입 같은 nested type을 인식하지 못한다.

### 2안: Prometheus Pushgateway 기반

카카오의 사례처럼 Prometheus Pushgateway를 통해 간접 연동하는 방식이다. 메타테이블 쿼리를 배치로 실행하고 결과를 Pushgateway에 밀어넣으면 Prometheus가 주기적으로 가져간다.

장점이 확실했다.

- Trino에 대한 불필요한 중복 쿼리가 없다.
- 메트릭 계산 로직을 자유롭게 커스터마이징할 수 있다.
- Trino의 제약을 우회하기 위해 Spark를 쓰거나 Glue Catalog API를 직접 활용할 수도 있다.
- Airflow의 컴팩션 작업 후행으로 메트릭 푸시를 붙이면 자연스럽게 파이프라인이 구성된다.

결론은 1안으로 빠르게 시작하되 2안으로 점진적으로 전환하는 방향이었다. 최종적으로 두 가지를 병행해서 용도에 따라 사용한다.

---

## 대시보드 구성

### 전체 테이블 헬스 모니터링

Pushgateway 기반 대시보드다. 모든 Iceberg 테이블의 상태를 시간 흐름에 따라 추적한다.

`$files` 메타테이블은 가장 최신 스냅샷이 참조하는 파일만 보여준다. 과거 스냅샷의 파일 정보까지 포함한 히스토리는 제공하지 않는다. 하지만 메타테이블 쿼리 결과를 주기적으로 Pushgateway에 밀어넣고 Prometheus에서 가져가면, 특정 시점의 스냅샷 상태가 시계열 메트릭으로 쌓인다. 시간에 따른 변동 추이를 볼 수 있게 되는 거다.

테이블 수가 많아지면 각 테이블의 지표 스케일이 달라서 한 화면에서 직관적으로 보기 어렵다. 정규식 텍스트 검색 옵션을 추가해서 특정 테이블만 필터링할 수 있도록 했다.

### 특정 테이블 상세 탐색

Grafana의 Trino 데이터소스를 직접 활용하는 대시보드다. 특정 테이블을 선택하면 현 시점 기준으로 `$partitions` 메타테이블을 조회해서 각 날짜 파티션의 파일 수와 파일 크기를 보여준다.

전체 테이블 대시보드는 시간 흐름에 따른 추이를 보는 용도고, 이 대시보드는 특정 시점의 파티션별 분포를 보는 용도다.

---

## Trino 메타테이블 버그와 체리피킹

모니터링 구축 당시 Trino v451을 운영하고 있었다. 두 가지 문제를 발견해서 상위 버전의 픽스를 체리피킹했다. 이후 v476으로 업그레이드하면서 이 픽스들은 자연스럽게 포함됐다.

### $files 메타테이블의 delete file 누락

Iceberg v2의 delete file(positional, equality)을 `$files` 메타테이블이 인식하지 못하는 버그가 있었다. v455에서 수정됐다. 포크 레포의 stage 브랜치에 해당 커밋을 체리피킹해서 적용했고, 정상적으로 delete file 통계가 조회되는 것을 확인했다.

### $files 메타테이블에 partition 필드 없음

v451까지는 `$files` 메타테이블에 각 파일이 속한 파티션 정보가 없었다. 파티션별 파일 통계를 수집하려면 이 필드가 필수적이다. v465에서 `partition`, `spec_id`, `sort_order_id`, `readable_metrics` 컬럼이 추가됐다. 이것도 체리피킹해서 적용했다.

---

## grafana-trino 플러그인의 한계

Grafana의 Trino 플러그인은 nested type과 ROW 타입 필드를 인식하지 못한다. `$partitions` 메타테이블의 ROW 타입 `partition` 필드가 대표적인 예다. `partition.log_ts_day`처럼 명시적으로 지정해줘야 한다.

이 때문에 시간 파티션 외에 추가 파티션이 있는 테이블이나 day 단위가 아닌 파티션을 가진 테이블을 범용적으로 커버하기 어렵다. 플러그인 자체의 근본적인 제약이다.

---

## 무거운 테이블의 메타 쿼리 성능

앱 로그나 웹 로그 같은 대용량 테이블의 메타테이블 쿼리는 매우 느렸다. 현업 적용이 불가능할 정도였다. 해당 테이블들은 모니터링 대상에서 일단 제외했다.

이 테이블들은 규모도 크지만 `rewrite_manifests()` 최적화가 제대로 수행되지 않고 있는 상태이기도 했다. 매니페스트 최적화가 되면 스캔 플래닝이 빨라져서 메타 쿼리 성능도 개선될 것으로 기대했다. 그런데 `rewrite_manifests()`를 정상화해도 메타 쿼리 성능에 유의미한 개선은 없었다.

### Spark vs Trino 메타테이블 쿼리 성능

Spark의 메타테이블 쿼리 성능이 Trino보다 월등히 좋다. 특히 무거운 테이블일수록 차이가 크다.

이유가 있다. Spark는 iceberg-core의 Table API를 통해 Snapshot에서 ManifestList로 필요한 DataFile만 조립한다. Trino는 메타데이터 파일들을 읽고 JOIN하는 방식이라 스캔 오버헤드가 크다.

배치성으로 Pushgateway에 메트릭을 밀어넣는 구성이라면 Trino 대신 Spark 엔진을 사용하는 것도 방법이다. Spark는 `all_files` 같은 메타테이블도 추가로 제공하니까 활용 범위가 더 넓다.

---

## 메트릭 파이프라인

### Pushgateway 배포

Prometheus Pushgateway를 Helm 차트로 EKS에 배포했다. Prometheus 서버에 Pushgateway 스크래퍼를 추가해서 메트릭을 수집하도록 구성했다.

### Airflow 배치 파이프라인

Airflow DAG으로 Iceberg 테이블 메트릭 푸시 파이프라인을 구성했다.

1. **대상 테이블 추출**: Glue API를 통해 Iceberg 포맷 테이블을 자동 추출한다. temp 스키마의 사용자 테이블은 제외하기 위해 스캔 대상 스키마 목록은 DAG 코드에서 관리한다.
2. **Dynamic task mapping**: 추출된 각 테이블에 대해 메타 쿼리를 수행하고 메트릭을 Pushgateway에 푸시하는 태스크가 동적으로 생성된다.
3. **메트릭 정의**: `iceberg_data_file_count` 같은 메트릭을 `env`, `partition`, `table` 라벨로 구분해서 수집한다.

컴팩션 작업의 후행으로 메트릭 푸시 로직을 붙이는 것도 가능하다. 기존 컴팩션 태스크 그룹 구현체에 후행 로직을 추가하는 방식이다.

### Iceberg 테이블 목록 자동 추출의 한계

모니터링 구축 초기(v451)에는 Trino에서 특정 스키마 내 Iceberg 포맷 테이블만 선별하는 것이 불가능했다. `SHOW TABLES` 쿼리나 `information_schema.tables` 조회 모두 Glue Catalog에 등록된 모든 테이블을 가져왔다. 테이블 리디렉션 비활성화 상태에서도 마찬가지였다. 그래서 Glue API를 통해 Iceberg 테이블을 추출하는 방식을 택했다.

v475에 추가된 `system.iceberg_tables` 테이블을 쓰면 Iceberg 테이블만 리스팅할 수 있다. 현재 운영 중인 v476에서 사용 가능하다.

---

## DELETE FILE 현황 파악

모니터링 구축 과정에서 CDC 테이블들의 delete file 생성 현황도 파악했다.

대부분의 CDC 싱크 테이블은 소스 RDB 자체가 히스토리성 테이블이라 사실상 append-only로 쌓이고 있다. 기존 레코드의 update가 드물게 발생하고 후행 컴팩션으로 정리되므로 delete file은 거의 쌓이지 않는다.

다만 직전에 추가된 레코드에 대해 바로 update/delete가 발생하는 일부 테이블에서는 positional delete file도 생성되고 있었다. 이런 테이블은 컴팩션 주기나 전략을 따로 검토할 필요가 있다.

---

## v451에서 v476까지: 모니터링 관련 개선 사항

모니터링 구축은 v451에서 시작했고, 이후 v476으로 업그레이드했다. 그 사이에 Iceberg 모니터링에 직접적으로 도움이 되는 개선 사항이 많았다.

| 버전 | 개선 내용 | 적용 방식 |
|------|----------|----------|
| v455 | `$files` 메타테이블에서 delete file 인식 버그 수정 | v451에 체리피킹 |
| v465 | `$files` 메타테이블에 `partition` 필드 추가 | v451에 체리피킹 |
| v466 | Glue Catalog 조회 병렬화로 성능 개선 | v476 업그레이드 시 반영 |
| v469 | `$all_entries` 메타테이블 추가 (Spark의 `all_entries`에 대응) | v476 업그레이드 시 반영 |
| v470 | `$all_entries` 버그 수정, `optimize_manifests` 프로시저 추가 | v476 업그레이드 시 반영 |
| v475 | `system.iceberg_tables` 테이블 추가 (Iceberg 테이블만 리스팅) | v476 업그레이드 시 반영 |

v455와 v465는 모니터링 구축에 필수적이어서 v451 시절에 체리피킹으로 먼저 적용했다. 나머지는 v476 업그레이드와 함께 자연스럽게 사용 가능해졌다.

---

## 배운 것

**메타테이블의 한계를 이해해야 한다.** `$files`는 최신 스냅샷만 보여준다. 히스토리를 보려면 주기적으로 결과를 외부에 저장하는 방법밖에 없다. Pushgateway가 딱 맞는 구성이다.

**Trino의 메타 쿼리는 무거운 테이블에서 느리다.** Spark가 훨씬 빠르다. 구현 방식 차이 때문이다. 배치 파이프라인이라면 Spark를 쓰는 것도 합리적이다.

**직접 조회와 배치 수집을 병행하자.** 전체 테이블 추이는 Pushgateway 기반 배치로, 특정 테이블 심층 분석은 직접 쿼리로. 두 가지를 용도에 맞게 나누니까 효과적이었다.

**Trino 포크를 운영한다면 체리피킹은 피할 수 없다.** 필요한 버그 수정이 상위 버전에만 있으면 가져와서 적용해야 한다. 특히 모니터링처럼 신뢰성이 중요한 영역에서는 부정확한 데이터가 더 위험하다.

**참고 자료:**
- [로그 유형별 Iceberg 테이블 적재 및 운영 전략 - kakao tech](https://tech.kakao.com/posts/694)
- [Iceberg Connector - Trino Documentation](https://trino.io/docs/current/connector/iceberg.html)
- [$files delete file bug fix (v455)](https://github.com/trinodb/trino/pull/23142)
- [$files partition field (v465)](https://github.com/trinodb/trino/pull/24102)
- [Glue catalog query parallelization (v466)](https://github.com/trinodb/trino/pull/24110)
- [$all_entries metadata table (v469)](https://github.com/trinodb/trino/pull/24543)
- [optimize_manifests procedure (v470)](https://github.com/trinodb/trino/pull/24678)
- [system.iceberg_tables (v475)](https://github.com/trinodb/trino/pull/25136)
- [Prometheus Pushgateway](https://github.com/prometheus/pushgateway)
- [grafana-trino plugin](https://github.com/trinodb/grafana-trino)
