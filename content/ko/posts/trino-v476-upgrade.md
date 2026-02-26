---
title: "Trino v451에서 v476으로: 25개 버전을 건너뛴 운영 클러스터 업그레이드"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [trino, upgrade, kubernetes, blue-green, parquet, iceberg, java]
showTableOfContents: true
summary: "Trino를 v451에서 v476으로 업그레이드했다. Blue/Green 배포로 OLAP 클러스터부터 점진 적용하는 과정에서 Materialized View 리그레션과 Parquet 읽기 리그레션을 발견했다. 바이너리 서치로 원인 버전을 v469로 좁히고, 커밋 단위 리버트로 원인 PR을 특정해서 커뮤니티에 이슈를 올렸더니 하루 만에 픽스가 나왔다."
---

Trino v451에서 v476으로 업그레이드를 진행했다. v476은 2025년 6월 5일에 릴리즈됐다. 25개 마이너 버전을 건너뛰는 업그레이드라서 변경사항이 방대했고, 실제로 배포 과정에서 두 건의 리그레션을 만났다. 그중 하나는 커뮤니티에도 보고된 적 없는 새로운 이슈였다.

이 글은 업그레이드에서 기대한 것, 실제로 겪은 것, 그리고 리그레션을 추적하고 해결한 과정을 정리한 기록이다.

---

## 왜 v476인가

### Java 22에서 Java 24로

Trino는 최신 JVM에 적극적으로 올라타는 프로젝트다. v447에서 Java 22를 요구하기 시작했고, v464에서 Java 23, v476에서 Java 24로 올라갔다. 단순히 JVM 버전만 올라가는 게 아니다. Project Hummingbird라는 이름의 성능 개선 프로젝트가 Java의 Vector API를 활용해서 Parquet 파일 읽기에 벡터화 디코딩을 적용하고 있다. v448에서 Parquet 벡터화 디코딩이 도입됐고, 256비트 이상의 벡터 레지스터가 필요해서 Graviton 2(r6g, m6g)에서는 비활성화되지만 Graviton 3 이상에서는 자동 활성화된다.

Java 24의 최신 JIT 컴파일러 최적화와 메모리 관리 개선이 쿼리 처리 성능에 직접 영향을 준다.

### S3 테이블 버킷 읽기 지원

v471부터 Iceberg 커넥터에서 S3 테이블 버킷에 대한 읽기가 가능해졌다. 매니지드 Iceberg 테이블인 S3 테이블 버킷의 도입을 검토하고 있었기 때문에, Trino에서 읽기를 지원하는 건 필수 전제 조건이었다. Glue Data Catalog의 Iceberg REST 엔드포인트를 통해 연동하며, 기존 glue 타입이 아닌 rest 타입 카탈로그를 별도 구성해야 한다.

### 버킷 파티션 푸시다운

v468에 추가된 파티셔닝 푸시다운이 있다. 버킷 파티션이 걸린 테이블에 대해 쿼리 조건을 파티션 레벨로 푸시다운해서 스캔 범위를 줄이는 최적화다.

우리 환경에서 앱 로그와 웹 로그 테이블이 `screen_name`에 버킷 파티션을 사용하고 있다. 이 테이블들은 규모가 크기 때문에 버킷 파티션 푸시다운의 효과가 클 것으로 기대했다.

### Iceberg 메타테이블 및 프로시저 개선

Iceberg 테이블 모니터링을 구축하면서 v451에 체리피킹했던 픽스들이 v476에 네이티브로 포함된다.

| 버전 | 개선 내용 |
|------|----------|
| v455 | `$files` 메타테이블에서 delete file 인식 버그 수정 |
| v465 | `$files` 메타테이블에 `partition` 필드 추가 |
| v466 | Glue Catalog 조회 병렬화로 성능 개선 |
| v469 | `$all_entries` 메타테이블 추가 (Spark의 `all_entries`에 대응) |
| v470 | `$all_entries` 버그 수정, `optimize_manifests` 프로시저 추가 |
| v475 | `system.iceberg_tables` 테이블 추가 (Iceberg 테이블만 리스팅) |

체리피킹 없이 이 기능들을 모두 쓸 수 있게 되는 것만으로도 업그레이드의 가치가 있었다.

### Alluxio 파일 시스템 캐시

기존에 사용하던 Alluxio 기반 로컬 디스크 캐시와 별개로, 외부 Alluxio 클러스터를 파일 시스템으로 연동하는 기능이 추가됐다. 카탈로그 간, 클러스터 간 캐시 공유가 가능해진다. 당장 적용할 계획은 아니지만 향후 캐시 아키텍처 개선에 활용할 수 있는 선택지다.

### Apache Ranger 플러그인

Apache Ranger 인가 플러그인이 Trino 공식 레포에 통합됐다. 기존에 별도 레포로 관리되던 것이 메인 프로젝트에 포함돼서 버전 호환성 관리가 수월해진다.

---

## 배포 전략

### Blue/Green 점진 배포

OLAP과 BI 두 클러스터 그룹 각각에 Blue/Green 쌍이 있다. 업그레이드 계획은 이랬다.

1. 각 클러스터 그룹의 한쪽(Green)에 먼저 배포
2. 1주일간 모니터링
3. 이상 없으면 나머지(Blue)에 적용

이슈 발생 시 Trino Gateway에서 해당 클러스터를 라우팅에서 즉시 빼고 롤백할 수 있다. 게이트웨이를 운영하고 있기 때문에 가능한 전략이다. 실제로 이 전략 덕분에 리그레션을 발견하고도 운영에 영향 없이 대응할 수 있었다.

OLAP-Green 클러스터부터 시작하기로 했다. BI보다 OLAP이 쿼리 패턴이 다양해서 리그레션을 더 빨리 발견할 수 있을 거라 판단했다.

### v451 때부터 이어진 미해결 이슈

v451 배포 때 문제가 됐던 이슈 두 가지가 여전히 해결되지 않은 상태였다. 업그레이드 전에 대응 방안을 준비해뒀다.

**워커 stuck 이슈**

v451 배포 당시 부하 상황에서 일부 워커가 멈추는 현상이 있었다. 원인은 `ThreadPerDriverTaskExecutor`라는 실험적 기능이었다. 별다른 문서화도 없이 디폴트로 활성화된 채 추가됐었다. 커뮤니티에 이슈가 올라와 있지만 아직 해결되지 않았다.

이번에도 동일하게 `experimental.thread-per-driver-scheduler-enabled=false`로 비활성화해서 대응할 계획이었다.

**파티션 프루닝 리그레션**

필터 조건에 형변환이 포함된 경우 파티션 프루닝이 동작하지 않는 이슈다. 예를 들어 `WHERE date_column = CAST('2024-01-01' AS DATE)` 같은 쿼리에서 파티션이 제대로 걸러지지 않는다.

v451 때는 리그레션을 유발한 커밋 자체를 리버트해서 대응했다. 그 사이에 Trino 커뮤니티에서 이 리그레션을 우회하는 escape hatch 설정이 추가됐다. unsafe pushdown을 허용하는 설정인데, 이번에는 이 설정을 적용해서 리그레션이 발생하지 않는지 확인하는 방향으로 진행했다.

---

## 배포 개시와 첫 번째 롤백

2025년 6월 23일, 코어타임 이후에 OLAP-Green 클러스터에 v476을 배포했다. 사용자들에게 Zeppelin 지원 채널을 통해 사전 공지를 했다.

### Materialized View REFRESH stuck

배포 직후 Airflow에서 Iceberg materialized view를 REFRESH하는 쿼리가 stuck 되고 있다는 제보가 들어왔다. 쿼리가 FINISHING 상태에서 끝나지 않고 멈춰 있었다.

Trino 레포를 확인하니 v475에 추가된 "Cleanup previous snapshot files during materialized view refresh"라는 변경이 리그레션을 유발한 것으로 확인됐고, 바로 전날 리버트 PR이 머지된 상태였다. 타이밍이 절묘했다. 해당 리버트 커밋을 포크 레포에 반영해서 이슈를 해소했다.

### Parquet 파일 데이터 미인식

더 심각한 이슈가 뒤따랐다. 사용자가 JupyterLab에서 쿼리 결과가 이상하다고 제보했다. 확인해보니 특정 Hive 테이블에서 데이터가 아예 조회되지 않았다.

문제의 패턴은 이랬다. 사용자가 Parquet 파일을 S3에 미리 적재해두고, 해당 경로를 바라보는 Hive 테이블을 나중에 생성하는 방식이었다. v451에서는 정상적으로 동작하던 패턴인데, v476에서는 테이블 생성 전에 올려둔 데이터 파일을 인식하지 못했다.

확인한 사항들:

- 데이터 파일을 어디서 생성했든(Trino v451, v476, Spark) 동일하게 재현
- Glue API 버전(v1, v2)과 무관
- Spark의 `MSCK REPAIR TABLE`이나 Trino의 `system.sync_partition_metadata()` 호출로도 파티션 데이터 인식 불가

이 이슈의 영향이 크다고 판단해서 **OLAP-Green을 v451로 롤백**했다. 원인을 파악한 뒤 재배포하기로 했다.

---

## 바이너리 서치로 원인 버전 특정

커뮤니티에 이 이슈가 보고된 적이 없었다. 직접 찾아야 했다.

v451과 v476 사이에 메타스토어와 Hive 커넥터에 대한 코드 변경이 매우 많았다. 릴리즈 노트만 훑어봐서는 원인을 특정할 수 없었다. 바이너리 서치 방식으로 접근했다.

Zeppelin에 테스트 노트북을 만들어서 각 버전에서 이슈 재현 여부를 체계적으로 확인했다.

```
v451 — 정상
  ↓ (중간 지점 테스트)
v463 — 정상
  ↓
v469 — 재현!
  ↓ (범위 좁힘)
v466 — 정상
v467 — 정상
v468 — 정상
v469 — 재현!
```

**v469에서 리그레션이 시작됨을 확인했다.**

다음은 v469의 Hive 커넥터 변경사항 중 어떤 커밋이 원인인지 찾는 작업이었다. v469 릴리즈 노트에서 Hive 커넥터 관련 변경사항을 확인하고, 해당 PR들을 하나씩 리버트하면서 이슈 재현 여부를 확인했다.

---

## 원인: Parquet Footer 최적화와 레거시 pyarrow

원인은 v469에 머지된 Parquet footer 파싱 최적화 PR이었다. 이 PR은 Parquet 파일의 footer 메타데이터를 읽을 때 `RowGroup.getFile_offset` 값을 신뢰하고 그 값을 기반으로 읽기 범위를 최적화하는 로직을 추가했다.

문제는 모든 Parquet 파일의 row group 오프셋이 정확하지는 않다는 점이었다. 이슈가 된 파일은 pandas를 통해 pyarrow v5.0.0(parquet-cpp-arrow v5.0.0)으로 작성된 Parquet 파일이었다. 이 레거시 버전에서 기록한 row group 오프셋에 오류가 있었다.

추가 테스트로 확인한 사항:

- 최신 pyarrow 버전으로 작성한 파일에서는 재현되지 않는다
- row group의 크기가 일정 규모 이상이 되어야만 오프셋 오류가 발생한다. 작은 파일에서는 재현되지 않았다
- v469 이전 버전에서는 `getFile_offset` 값을 사용하지 않았기 때문에 오프셋이 부정확해도 문제가 없었다

---

## 커뮤니티 이슈 제보와 하루 만의 픽스

원인 PR과 재현 방법을 정리해서 Trino 커뮤니티 Slack에 공유했다. "Incorrect results on parquet files written by parquet-cpp-arrow version 5.0.0"이라는 제목으로 이슈도 올렸다.

원래 최적화 PR의 작성자가 바로 근본 원인을 파악했다. 원인은 명확했다. 최적화 로직이 `RowGroup.getFile_offset` 값이 항상 정확하다고 가정하고 있었는데, 레거시 라이터가 부정확한 값을 기록하는 경우가 있었다.

픽스는 `RowGroup.getFile_offset` 대신 `ColumnChunk`의 first page offset을 사용하도록 변경하는 내용이었다. 방어 로직을 추가해서 오프셋 정보가 부정확한 파일에서도 정상 동작하도록 했다. 이슈 공유 후 하루 만에 픽스 PR이 올라왔고 v477 마일스톤에 머지됐다.

해당 픽스 커밋을 포크 레포에 체리피킹해서 스테이지에 배포하고 이슈 해소를 확인한 뒤, 운영에 반영했다.

---

## 재배포 및 결과

두 이슈를 모두 해결한 뒤 배포를 재개했다.

| 날짜 | 클러스터 | 작업 |
|------|---------|------|
| 6월 23일 | OLAP-Green | v476 최초 배포 → 이슈 발견, 롤백 |
| 6월 26일 오전 | OLAP-Green | 픽스 반영 후 재배포 |
| 6월 26일 오후 | OLAP-Blue | v476 배포 |

BI 클러스터 그룹은 금요일에 배포할 경우 주말 간 이슈 대응이 어렵다고 판단해서 별도 티켓으로 분리하고 다음 주 초에 진행했다.

배포 후 모니터링에서 클러스터 상태와 쿼리 실패 지표에 이상은 없었다. 사용자로부터 추가 이슈 제보도 없었다.

아직 충분한 기간은 아니지만, v476이 선배포된 클러스터의 쿼리 스루풋이 상대적으로 높은 양상이 확인됐다. 두 클러스터의 워커 수가 거의 동일한 상태에서 게이트웨이가 실행 쿼리 수 기반으로 라우팅한다는 점을 감안하면, v476 쪽이 더 많은 쿼리를 처리하면서도 부하가 낮았다는 뜻이다. 반대쪽에 헤비 쿼리가 몰린 영향일 수 있지만, 신규 버전의 전반적인 쿼리 성능이 개선됐다고 볼 여지가 있다.

---

## 배운 것

**Blue/Green 배포가 아니었으면 운영 장애가 됐다.** 한쪽 클러스터에 먼저 배포했기 때문에 리그레션을 발견해도 즉시 롤백하고 다른 쪽으로 트래픽을 돌릴 수 있었다. 25개 버전을 한 번에 올리는 메이저 업그레이드에서 Blue/Green은 선택이 아니라 필수다.

**바이너리 서치는 리그레션 디버깅의 정석이다.** 25개 버전 사이의 변경사항을 하나씩 확인하는 건 비현실적이다. 중간 버전에서 재현 여부를 확인하면서 범위를 절반씩 좁히면 5~6번의 테스트로 원인 버전을 특정할 수 있다. 원인 버전을 특정한 뒤에는 해당 버전의 관련 커밋을 하나씩 리버트하면서 원인 PR을 찾았다.

**잘 정리된 이슈 리포트는 빠른 해결로 이어진다.** 원인 PR, 재현 방법, 문제가 되는 파일의 특성(pyarrow 레거시 버전, row group 크기 조건)을 함께 공유하니까 원래 PR 작성자가 하루 만에 근본 원인을 파악하고 픽스를 작성해줬다. 이슈만 올리고 "안 돼요"라고 쓰는 것과 원인 분석까지 해서 공유하는 건 응답 속도에서 큰 차이가 난다.

**실험적 기능이 디폴트 활성화되는 걸 경계하자.** `ThreadPerDriverTaskExecutor`는 v451 때도 문제가 됐고 v476에서도 여전히 디폴트 활성화 상태였다. 커뮤니티 이슈가 열려 있는데도 해결이 안 되고 있다. 업그레이드 시 릴리즈 노트의 실험적 기능 관련 변경사항을 반드시 확인해야 한다.

**이전 업그레이드의 교훈을 체크리스트로 남겨두자.** v451 때 발생한 두 이슈(워커 stuck, 파티션 프루닝 리그레션)에 대한 대응 방안을 미리 준비해뒀기 때문에 이번에는 빠르게 대응할 수 있었다. 업그레이드마다 겪은 이슈와 대응 방안을 기록해두면 다음 업그레이드가 수월해진다.

**참고 자료:**
- [Trino v475 Release Notes](https://trino.io/docs/current/release/release-475.html)
- [Trino v476 Release Notes](https://trino.io/docs/current/release/release-476.html)
- [Require Java 24 (v476)](https://github.com/trinodb/trino/pull/25815)
- [Add partitioning push down (v468)](https://github.com/trinodb/trino/pull/23432)
- [Add Apache Ranger authorizer plugin](https://github.com/trinodb/trino/pull/22675)
- [Alluxio file system support](https://trino.io/docs/current/object-storage/alluxio.html)
- [Add read support for S3 Tables in Iceberg (v471)](https://github.com/trinodb/trino/pull/24815)
- [ThreadPerDriverTaskExecutor issue](https://github.com/trinodb/trino/issues/21512)
- [Partition pruning regression](https://github.com/trinodb/trino/issues/22268)
- [Escape hatch for unsafe pushdowns](https://github.com/trinodb/trino/pull/22987)
- [MV refresh revert PR](https://github.com/trinodb/trino/pull/26051)
- [Parquet footer optimization (v469, regression cause)](https://github.com/trinodb/trino/pull/24618)
- [Parquet file_offset issue report](https://github.com/trinodb/trino/issues/26058)
- [Parquet fix PR (v477)](https://github.com/trinodb/trino/pull/26064)
