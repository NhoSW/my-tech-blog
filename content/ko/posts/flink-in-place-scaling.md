---
title: "Flink on EKS에서 In-place 스케일링 적용기: 재시작 없이 TaskManager를 늘리고 줄이기"
date: 2026-03-03
draft: false
categories: [Data Engineering]
tags: [flink, kubernetes, eks, autoscaling, spot-instance, kafka, streaming]
showTableOfContents: true
summary: "추천시스템의 Flink 어플리케이션은 지연시간 1분 미만이 요구되어 오토스케일링과 스팟 인스턴스를 적용하지 못하고 있었다. 오토스케일링이나 스팟 회수로 Flink 앱이 재시작되면 실제 처리 시작까지 2~3분이 걸리기 때문이다. Flink 1.18의 adaptive scheduler와 K8s operator 1.8을 활용해 in-place 스케일링을 적용하고, 스케일링 시 컨슈머 랙을 1/5로 줄인 과정을 정리한다."
---

추천시스템에서 Flink on EKS를 운영하고 있었다. 추천 응답 지연시간이 1분 미만이어야 하는 요구사항 때문에, 오토스케일링과 스팟 인스턴스를 적용하지 못하고 온디맨드 인스턴스를 고정 할당으로 운영하고 있었다. 비용이 상당했다.

문제의 핵심은 Flink 어플리케이션의 재시작 시간이었다. 오토스케일링이나 스팟 회수로 앱이 재시작되면, 체크포인트 복구와 카프카 컨슈머 그룹 재구성을 거쳐 실제 처리를 시작하기까지 2~3분 이상이 걸린다. 1분 미만의 지연 요구사항을 충족할 수 없었다.

Flink에서 재시작 없이 스케일링할 수 있는 방법이 있는지 찾아보기로 했다.

---

## 두 가지 접근: Reactive Scheduler와 Adaptive Scheduler

Flink에서 in-place 스케일링을 지원하는 방식은 두 가지가 있다.

**Reactive Scheduler**는 Flink K8s Operator 1.2에서 도입되었다. TaskManager 파드 수 변화를 감지해서 자동으로 잡의 parallelism을 조절한다. 하지만 standalone 모드에서만 동작한다는 제약이 있다.

실제로 standalone 모드를 테스트해봤는데, JobManager가 시작 시점에 JAR 파일을 찾지 못하는 문제가 발생했다.

```
org.apache.flink.util.FlinkException: An error occurred while access the provided classpath.
Caused by: java.nio.file.NoSuchFileException: /opt/flink/usrlib/ds-stream-assembly.jar
```

native 모드와 동일한 도커 이미지, 동일한 JAR 경로를 사용했는데 standalone에서만 파일을 찾지 못했다. standalone 모드에서는 `StandaloneApplicationClusterEntryPoint`가 classpath를 다르게 해석하는 것으로 보였다. 빠르게 해결이 어려워 보여서 다른 방향으로 전환했다.

**Adaptive Scheduler**는 Flink 1.18에서 도입된 방식이다. 잡 그래프 상의 각 vertex의 parallelism을 REST API로 조절할 수 있고, Flink K8s Operator 1.8에서 오토스케일러와 연동된다. standalone 모드 제약이 없다. 이쪽으로 테스트를 진행하기로 했다.

---

## Flink K8s Operator 업그레이드: 1.5 → 1.8

adaptive scheduler 기반의 in-place 스케일링을 사용하려면 Flink K8s Operator 1.8이 필요했다. 기존 1.5에서 1.8로 업그레이드했다.

Flink 자체도 1.17에서 1.18로 업그레이드해야 했다. 여기서 빌드 문제가 두 가지 발생했다.

### jackson-module-scala 버전 충돌

```
com.fasterxml.jackson.databind.JsonMappingException:
Scala module 2.13.2 requires Jackson Databind version >= 2.13.0 and < 2.14.0
- Found jackson-databind version 2.14.3
```

`jackson-module-scala`는 `jackson-databind`와 major/minor 버전이 일치해야 한다. 그런데 `jackson-databind` 2.14.3이 어디선가 로드되어 충돌이 발생했다.

`sbt dependencyTree`로 확인해도 2.14.3이 직접 의존성에 나타나지 않았다. Flink 관련 라이브러리가 내부적으로 참조하는 것으로 보였는데, 정확한 경로는 끝내 파악하지 못했다. Flink 코어 라이브러리를 1.17로 원복하면 문제가 사라지는 것으로 보아, 1.18에서 의존하는 jackson 버전이 변경된 것이 원인이었다.

결국 모든 jackson 라이브러리 버전을 2.14.3으로 통일해서 해결했다.

```scala
val jacksonVersion = "2.14.3"   // Only 2.14.x is compatible with Flink 1.18.x
```

### ExecutionConfig ClassNotFoundException

jackson 문제를 해결하자 다른 에러가 나타났다.

```
Cause: java.lang.ClassNotFoundException: org.apache.flink.api.common.ExecutionConfig
```

`ExecutionConfig`은 `flink-core`에 포함된 클래스로, Flink 초기 버전부터 항상 존재하는 클래스다. 의존성 문제가 아니라 ClassLoader가 꼬인 것으로 보였다.

Flink 메일링 리스트에서 동일한 현상을 발견했다. sbt가 테스트를 같은 JVM에서 실행할 때 ClassLoader 격리가 제대로 되지 않는 문제였다. 프로세스를 fork해서 별도 JVM에서 테스트를 실행하면 해결된다.

```scala
Test / fork := true
```

sbt 문서에서도 "역직렬화 및 클래스 로딩이 예상과 다르게 동작할 수 있다"고 언급하고 있지만, 그 이상의 설명은 없었다.

---

## 수동 테스트: Scale 버튼

베타 환경에서 Flink 1.18.1과 K8s Operator 1.8을 배포한 후 먼저 수동 테스트를 진행했다.

Flink 1.18에서는 Flink Web UI에 Scale 버튼이 추가되었다. 이 버튼을 누르면 각 vertex의 parallelism을 조절할 수 있다. REST API로도 동일한 조작이 가능하다.

Scale 버튼을 눌러봤더니, 실제로 잡의 재시작 없이 새로운 TaskManager 파드가 생성되어 바로 실행되는 것을 확인했다. 수동 in-place 스케일링은 정상 동작했다.

문제는 이것을 메트릭 기반으로 자동화하는 것이었다. K8s Operator 1.8 문서에 따르면 오퍼레이터의 오토스케일러가 이를 자동으로 처리한다고 되어 있었다. 베타 환경에서는 충분한 트래픽이 없어서 오토스케일링을 트리거할 수 없었기 때문에, 스테이지 환경에서 테스트하기로 했다.

---

## 오토스케일링 테스트: memory autotuning의 함정

스테이지 환경에서 두 개의 Flink 앱을 띄우고 `consumer.offset=earliest` 옵션으로 대량 트래픽을 발생시켰다. TM 1개로 시작했더니 오토스케일러가 컨슈머 랙을 감지하고 TM을 각각 120개, 64개로 스케일 아웃했다.

그런데 in-place가 아니라 1.17과 동일하게 잡 전체가 재시작됐다.

원인을 찾아보니 Operator 1.8에서 새로 추가된 **memory autotuning** 기능이 문제였다. 이 기능을 같이 활성화해두었는데, 오토스케일링이 발생할 때마다 메모리 설정까지 재조정하면서 전체 재시작이 발생한 것이다.

memory autotuning을 비활성화한 후 다시 테스트했다. 그래프에서 차이가 명확했다.

- **비활성화 전**: 오토스케일링 시 TM 수가 0으로 떨어졌다가 올라오면서 처리 공백이 발생
- **비활성화 후**: TM 수가 점진적으로 변경되면서 처리 공백 없음

in-place 스케일링이 정상 동작하는 것을 확인했다.

---

## 성능 비교: 컨슈머 랙 기준

in-place 스케일링이 실제로 얼마나 개선되는지 정량적으로 비교해봤다.

### In-place 비활성 (기존 방식)

스케일링 발생 시 전체 앱 재시작이 일어나면서 TM 수가 일시적으로 0이 된다. 이 과정에서 카프카 컨슈머 랙이 피크를 형성한다.

- 컨슈머 랙 최대값: **130만~160만**
- 피크 지속 시간: **5~7분**

### In-place 활성 (개선 후)

스케일링 시에도 TM 수가 0으로 떨어지지 않는다. 다만 카프카 컨슈머 rebalancing은 여전히 발생하고, 이 과정에서 각 컨슈머가 연결을 끊고 다시 맺는 동안 약간의 지연이 생긴다.

- 컨슈머 랙 최대값: **25만~50만**
- 피크 지속 시간: **2~3분**

| 지표 | In-place 비활성 | In-place 활성 | 개선 |
|------|----------------|--------------|------|
| 컨슈머 랙 최대값 | 130만~160만 | 25만~50만 | ~1/5 |
| 피크 지속 시간 | 5~7분 | 2~3분 | ~1/2 |

기대했던 것처럼 "지연 제로"는 아니었다. in-place 스케일링이 Flink 잡 재시작을 방지하더라도, 카프카 컨슈머 그룹의 rebalancing 과정은 피할 수 없다. 하지만 랙 최대값이 1/5로 줄고 지속 시간이 절반으로 줄어든 것은 의미 있는 개선이다.

---

## 노드 할당 전략 개선

in-place 스케일링이 가능해지면서 노드 할당 전략도 변경할 수 있게 되었다.

기존에는 스팟 인스턴스 회수 시 전체 앱이 재시작되는 것을 우려해 복잡한 구조를 운영하고 있었다. 스팟과 온디맨드를 동시에 사용하되 스팟을 우선 배치하고, 온디맨드에 할당된 파드를 주기적으로 eviction하고, 스팟이 없을 때 노드가 뜨는 시간을 벌기 위한 over-provisioning 파드까지 운영하고 있었다.

in-place 스케일링으로 TM 파드의 생사가 잡 전체 재시작을 의미하지 않게 되면서, 이 구조를 단순화할 수 있었다.

**변경된 전략:**

- **JobManager**: 항상 온디맨드 노드에 할당. JM이 죽으면 앱 전체가 재시작되므로, 스팟 회수에 노출시키지 않는다.
- **TaskManager**: 스팟 우선, 온디맨드 폴백. TM이 회수되어도 in-place 스케일링으로 나머지 TM이 처리를 이어받는다.
- **Over-provisioning 파드**: 제거. JM이 온디맨드에서 안정적으로 실행되므로 앱 전체 재시작이 배포 시를 제외하면 거의 없다. 새 노드가 뜨는 데 시간이 걸려도 크게 문제되지 않는다.
- **Eviction 로직**: JM과 TM 모두 eviction하던 것에서 TM만 eviction하도록 단순화.

### 스팟 종료 지연은 불필요

처음에는 Trino worker에서 적용했던 방식 — 스팟 회수 시 신규 파드가 준비될 때까지 기존 파드의 종료를 지연시키는 것 — 을 Flink에도 적용하려고 했다. 하지만 이 방식을 적용하더라도 카프카 컨슈머 rebalancing 과정에서의 2~3분 지연은 동일하게 발생한다. in-place 스케일링이 이미 같은 수준의 효과를 제공하므로 별도 적용이 필요 없다고 판단했다.

---

## 배운 것

**in-place 스케일링은 "재시작 없음"이지 "지연 없음"이 아니다.** Flink 잡 재시작은 방지되지만 카프카 컨슈머 rebalancing은 피할 수 없다. 스케일링 시 컨슈머들이 파티션을 재분배하는 동안 2~3분의 처리 지연이 발생한다. 기대치를 정확히 설정하는 것이 중요하다.

**새 기능을 여러 개 동시에 켜면 원인 파악이 어렵다.** memory autotuning과 in-place 스케일링을 함께 활성화했더니 in-place가 동작하지 않았다. 새 기능은 하나씩 활성화하고 각각의 효과를 확인해야 한다.

**문제를 풀기 전에 문제의 구조를 먼저 파악하자.** 스팟 회수 시 기존 파드 종료를 지연시키는 방식(Trino에서 효과적이었던)을 Flink에도 적용하려 했지만, Flink의 병목은 파드 종료가 아니라 컨슈머 rebalancing이었다. 같은 해법이 다른 시스템에도 통하리라는 가정은 위험하다.

**복잡한 운영 구조는 근본 원인이 해결되면 단순화할 수 있다.** over-provisioning, 이중 eviction, 스팟/온디맨드 혼합 로직은 "재시작이 비싸다"는 전제 위에 쌓인 것이다. 재시작 비용이 줄어들자 이 구조들이 불필요해졌다. 근본 원인을 해결하면 파생된 복잡성이 함께 사라진다.

**참고 자료:**
- [Flink K8s Operator 1.2: Improved Upgrade Flow](https://flink.apache.org/2022/10/07/apache-flink-kubernetes-operator-1.2.0-release-announcement/)
- [Flink K8s Operator 1.8: In-place Scaling Support](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-release-1.8/docs/custom-resource/autoscaler/)
- [Flink Mailing List: ClassNotFoundException with Flink 1.18](https://www.mail-archive.com/user@flink.apache.org/msg52040.html)
- [sbt: Running Project Code](https://www.scala-sbt.org/1.x/docs/Running-Project-Code.html)
