---
title: "ALB 로그를 Iceberg 테이블로 만들기까지"
date: 2026-02-25
draft: false
categories: [Data Engineering]
tags: [alb, filebeat, flink, iceberg, sqs, kafka, kubernetes]
showTableOfContents: true
summary: "S3에 CSV로 쌓이는 ALB 로그를 Iceberg 테이블로 바꾸는 시스템을 구축했다. Flink filesystem connector의 메모리 한계를 겪고 filebeat + SQS + Kafka 구조로 재설계한 과정, 오토스케일링 적용, 체크포인트 튜닝, 그리고 운영 중 마주친 여러 삽질을 정리했다."
---

클라우드 인프라팀에서 요청이 왔다. 장애 분석할 때 ALB 로그를 테이블로 조회할 수 있게 해달라고.

ALB 로그는 S3에 CSV 형태로 쌓이고 있었다. 필요할 때마다 Athena로 직접 조회했는데 느리고 불편했다. 텍스트 포맷이니 조회 성능에도 한계가 있었다. Iceberg로 Parquet 파일로 변환해서 적재하면 실시간 대시보드까지 만들 수 있다.

이 글은 그 시스템을 구축하면서 겪은 시행착오를 정리한 것이다.

---

## 처음 설계: Flink filesystem connector

가장 단순한 구조로 시작했다.

```
S3 파일 → Flink (filesystem connector) → Iceberg 테이블
```

Flink의 filesystem connector가 S3 버킷을 감시하다가 새 파일이 들어오면 읽어서 각 라인을 레코드로 방출한다. 거기에 Iceberg sink를 붙이면 끝이다.

문제는 **메모리**였다. filesystem connector는 이미 처리한 파일 목록 전체를 Flink state에 들고 있는다. ALB 로그가 하루에도 수만 개씩 쌓이니 시간이 갈수록 state가 불어나서 메모리가 터졌다. 확장의 한계에 부딪혀서 구조를 바꿔야 했다.

---

## 현재 구조: S3 → SQS → Filebeat → Kafka → Flink → Iceberg

파일 감시를 Flink에서 분리했다. 메시지 큐와 경량 수집기를 앞단에 두는 구조다.

```
S3 버킷 → SQS (파일 이벤트) → Filebeat → Kafka → Flink → Iceberg
```

각 구간이 하는 일은 이렇다.

1. **S3 → SQS**: 버킷 설정으로 ALB 로그 파일이 추가될 때마다 SQS에 해당 파일 경로가 메시지로 들어간다
2. **SQS → Filebeat → Kafka**: Filebeat가 SQS에서 메시지를 읽고 S3 파일을 직접 가져와 각 라인을 Kafka 토픽으로 보낸다
3. **Kafka → Flink → Iceberg**: Flink가 Kafka에서 레코드를 읽고 CSV 파싱 후 가공(파일 경로에서 계정번호, 서비스명, 역할 추출)하여 Iceberg 테이블에 쓴다

Flink는 Kafka만 바라보면 된다. 파일 목록을 state에 들고 있을 필요가 없어졌다.

---

## Filebeat에 정착하기까지

Filebeat를 선택하기 전에 Kafka Connect Filesystem Connector를 먼저 시도했다. 결과적으로 실패했고 그 과정에서 배운 것이 있다.

### Kafka Connect Filesystem Connector — 포기

써드파티 커넥터였는데 문제가 여러 개 있었다.

- Maven에 올라가 있지 않아서 직접 빌드해야 했다
- GitHub 마지막 커밋이 4년 전이었고 Java 8 기반이었다
- 같은 메시지가 두 번 이상 중복 발행되는 현상이 발생했다
- S3 파일 move(클린업)가 정상적으로 동작하지 않았다. `FileUtil.copy()`가 S3A 파일시스템에서 제대로 안 돌아가는 문제였다

커뮤니티에서 충분히 검증되지 않은 도구를 억지로 쓰는 것보다 잘 관리되는 도구를 쓰는 게 낫다고 판단했다.

### Filebeat 버킷 리스팅 모드 — 스케일링 한계

Filebeat는 S3 input을 두 가지 모드로 지원한다. 처음에는 **버킷 리스팅 모드**를 썼다. 주기적으로 S3를 스캔해서 새 파일을 감지하는 방식이다.

동작은 잘 됐는데 수평 스케일링이 안 됐다. 여러 Filebeat 인스턴스를 띄우면 같은 파일을 중복 처리한다. 각 인스턴스가 서로의 존재를 모르기 때문이다. 공식 문서에도 버킷 리스팅 모드에서는 single instance + 수직 스케일링만 하라고 명시되어 있다.

또 다른 문제도 있었다. 처리 완료된 파일을 백업 버킷으로 옮기는 기능(`backup_to_bucket_arn`)이 대부분의 Filebeat 버전에서 버킷 리스팅 모드와 제대로 동작하지 않았다. GitHub Issue를 찾아보니 fix가 머지됐다가 실수로 롤백된 상태였다. 8.15.3 버전에서만 정상 동작을 확인했다.

### SQS 모드로 전환

SQS 모드는 수평 스케일링이 가능하다. SQS가 메시지 분배를 해주기 때문에 여러 Filebeat 파드가 서로 겹치지 않고 처리할 수 있다. S3 API 호출 비용도 줄어든다.

단점은 기존 파일을 소급 처리하지 못한다는 점이다. SQS에 이벤트가 들어온 시점 이후의 파일만 처리한다. 필요하면 기존 파일에 대해 수동으로 SQS 이벤트를 발행하면 된다. ALB 로그 상황에서는 문제가 안 됐다.

한 가지 더. SQS 모드에서는 `backup_to_bucket_arn` 설정을 아예 사용할 수 없다. config 검증 단계에서 에러가 난다. 생각해 보면 SQS를 쓰면 새로 추가된 파일과 기존 파일을 구분할 필요가 없으니 백업 기능이 필요 없는 게 맞다.

최종 Filebeat 설정은 이렇다.

```yaml
filebeat.inputs:
  - type: aws-s3
    queue_url: https://sqs.ap-northeast-2.amazonaws.com/xxxx/s3-elb-logs-events
    file_selectors:
      - regex: ".*\\.log\\.gz"
    number_of_workers: 64
    decompress: true
    codec.line:
      format: message
output.kafka:
  hosts:
    - kafka-cluster:9092
  topic: cloudinfra.prod.streaming.alb-access-log.csv
  codec.format:
    string: '%{[aws.s3.object.key]} %{[message]}'
  compression: zstd
```

핵심은 `codec.format`에서 S3 파일 경로(`aws.s3.object.key`)를 메시지 앞에 붙여서 보내는 부분이다. Flink에서 이 경로를 파싱해 계정번호와 서비스명을 추출한다.

---

## 오토스케일링

### Filebeat: SQS 기반 KEDA 스케일링

SQS의 Visible Messages 수를 기준으로 KEDA ScaledObject를 구성했다.

```yaml
spec:
  cooldownPeriod: 300
  maxReplicaCount: 128
  minReplicaCount: 1
  pollingInterval: 30
  triggers:
    - type: aws-sqs-queue
      metadata:
        queueLength: '50'
        queueURL: https://sqs.ap-northeast-2.amazonaws.com/xxxx/s3-elb-logs-events
```

처음 적용했을 때 KEDA operator의 IAM Role 권한 문제로 SQS 메트릭 조회가 실패했다. ScaledObject에서 `identityOwner: operator`로 설정하면 KEDA operator가 자기 role로 직접 SQS를 조회한다. 이 role에 `sqs:GetQueueAttributes` 권한을 추가해서 해결했다.

### Flink: K8s Operator 오토스케일링

Flink는 Kubernetes Operator의 내장 오토스케일링을 사용한다.

```yaml
job.autoscaler.target.utilization: "0.75"
job.autoscaler.target.utilization.boundary: "0.15"
pipeline.max-parallelism: '480'
```

---

## Flink 체크포인트 튜닝

체크포인트 간격을 1분에서 5분으로 늘렸더니 연쇄적으로 문제가 터졌다.

### Heartbeat timeout

체크포인트가 오래 걸리면서 TaskManager의 heartbeat가 타임아웃됐다. 기본값이 너무 짧았다.

```yaml
heartbeat.interval: '60000'
heartbeat.timeout: '300000'
pekko.ask.timeout: 10m
```

### OOM: Java heap space

체크포인트 간격이 길어지면서 Kafka fetch 버퍼가 힙에 쌓이는 양이 늘었다. 파드 메모리를 2G → 4G → 6G로 올려도 같은 에러가 반복됐다.

원인은 JVM Task Heap 메모리 할당이 부족한 것이었다. 파드 전체 메모리는 여유가 있었지만 Task Heap은 2.32GB만 할당되어 꽉 찼다. `taskmanager.memory.task.heap.size`를 직접 지정해서 해결했다.

```yaml
taskmanager.memory.task.heap.size: 4608m
taskmanager.memory.managed.size: 512m
taskmanager.memory.network.fraction: '0.02'
```

### 최종 설정

여러 차례 튜닝 끝에 체크포인트 간격을 10분까지 늘릴 수 있었다.

```yaml
execution.checkpointing.interval: 10m
execution.checkpointing.timeout: 5m
heartbeat.interval: '60000'
heartbeat.timeout: '300000'
pekko.ask.timeout: 10m
taskmanager.memory.task.heap.size: 4608m
taskmanager.memory.managed.size: 512m
taskmanager.memory.network.fraction: '0.02'
```

---

## 컴팩션 실패와 Upsert 제거

체크포인트 간격을 10분으로 바꾼 뒤에도 Iceberg 컴팩션이 계속 실패했다. 원인으로 추정되는 요소를 전부 제거하기로 했다.

1. **새 Iceberg 테이블을 만들었다.** 기존 테이블의 메타데이터가 오염(?)된 상태일 수 있어서 깨끗한 테이블로 교체했다.
2. **Upsert 로직을 제거했다.** 원래 중복 방지를 위해 UPSERT 모드로 적재하고 있었는데 이게 컴팩션 충돌의 원인이었다.

Upsert를 빼면 중복 데이터가 생길 수 있다. 확인해 보니 중복이 있긴 했는데 적재 과정에서 생긴 게 아니라 **원본 ALB 로그 자체에 중복**이 있었다. 실질적으로 문제없다고 판단하고 append-only로 전환했다. 이후 컴팩션 작업이 빠르게 성공했다.

---

## 운영 중 마주친 사건: ALB 로그 컬럼 수 변경

어느 날 새벽 4시 무렵부터 Flink 앱의 CSV 파싱이 실패하기 시작했다. 원인은 ALB 로그 컬럼이 31개에서 34개로 늘어난 것이었다. AWS에서 사전 공지 없이 변경한 거였다.

### 첫 번째 시도: DLQ (Dead Letter Queue)

csv-dlq 포맷을 적용해서 파싱 에러가 난 레코드를 DLQ 토픽으로 보내도록 했다. 앱은 돌아왔지만 새로운 문제가 생겼다. 포맷 에러가 TaskManager 로그에 기록되면서 **ephemeral-storage**가 가득 찼다. 파드가 Evicted 상태로 죽었다 살아났다를 반복했다.

```
The node was low on resource: ephemeral-storage.
```

log4j 설정을 수정해서 포맷 에러 로그 레벨을 ERROR로 올려 해결했다.

```properties
logger.format.name = com.woowahan.dataservice.format
logger.format.level = ERROR
```

### 두 번째 문제: DLQ 토픽 과부하

DLQ 토픽에 메시지가 너무 많이 들어가면서 Kafka 클러스터에 부하가 생겼다. 결국 DLQ를 롤백하고 `csv.ignore-parse-error` 옵션으로 파싱 에러를 무시하도록 바꿨다. 소스 테이블에 dummy 컬럼 3개를 추가해서 31개든 34개든 모두 수용할 수 있게 처리했다.

---

## 중복 방지: UPSERT와 Primary Key

초기에는 중복 방지를 위해 Iceberg UPSERT를 사용했다. PK(Primary Key)를 잡아야 하는데 ALB 로그에는 고유 식별자가 없다.

46만 건의 로그를 조사해서 `file_path`, `time`, `http_request`, `client_addr`, `target_addr`, `request_creation_time` 5개 필드 조합이 거의 유니크하다는 걸 확인했다. 실제 중복은 전체에서 1건뿐이었고 그 2개 레코드는 모든 필드가 동일해서 사실상 구분 불가능한 로그였다.

Iceberg 테이블에서 PK를 지정하려면 Flink SQL의 `SET IDENTIFIER FIELDS`를 써야 했다. Spark SQL에서는 PRIMARY KEY 문법을 지원하지 않고 Flink SQL에서는 bucketing과 hidden partitioning을 지원하지 않아서 테이블 생성에 애를 먹었다.

결국 앞서 설명한 대로 컴팩션 안정성을 위해 UPSERT를 제거했다.

---

## 모니터링

### 핵심 지표

| 지표 | 의미 |
|------|------|
| SQS Approximate Number Of Messages Visible | 처리 대기 중인 파일 수 |
| SQS Approximate Number Of Messages Not Visible | 현재 Filebeat가 처리 중인 파일 수 |
| Kafka consumer lag | Flink의 처리 지연 |
| Flink job status | 앱 실행 상태 |

### 알럿 설정

- SQS: Visible Messages가 임계치 초과 시 (밀림 감지)
- Filebeat: CPU/메모리 사용률 과다, 파드 미실행
- Flink: job이 non-running 상태 전환 시

---

## 정리

돌아보면 가장 큰 교훈은 세 가지다.

**Flink에 파일 감시를 맡기지 말 것.** Filesystem connector는 처리한 파일 목록을 state에 들고 있어서 장기 운영하면 메모리가 터진다. 파일 감시는 SQS + Filebeat 같은 외부 도구에 맡기고 Flink는 스트림 처리에만 집중시키는 게 맞다.

**Upsert는 컴팩션과 충돌한다.** Iceberg CDC 테이블의 position delete 문제다. 로그성 데이터라면 append-only가 운영 안정성 면에서 훨씬 낫다. 원본 데이터 자체의 중복은 소비자 쪽에서 처리하면 된다.

**AWS는 사전 공지 없이 포맷을 바꿀 수 있다.** ALB 로그 컬럼이 갑자기 늘어나는 일이 실제로 일어났다. CSV 파싱을 하는 파이프라인이라면 `ignore-parse-error` 같은 방어 로직이 필수다.

**참고 자료:**
- [Filebeat AWS S3 Input](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-input-aws-s3.html)
- [Flink Filesystem Connector](https://nightlies.apache.org/flink/flink-docs-release-1.19/docs/connectors/table/filesystem/)
- [Apache Iceberg - Flink Writes](https://iceberg.apache.org/docs/latest/flink-writes/)
- [KEDA AWS SQS Queue Scaler](https://keda.sh/docs/2.12/scalers/aws-sqs/)
