---
title: "Kafka Rack Awareness와 Spark: 현재 지원하지 않는다"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [kafka, spark, rack-awareness, kubernetes, aws, networking]
showTableOfContents: true
summary: "Kafka 클러스터의 rack awareness를 Spark에도 적용해서 cross-AZ 네트워크 비용을 줄이려 했다. AZ 정보 추출까지는 해결했지만, Spark 자체가 Kafka 연동 시 rack awareness를 지원하지 않았다. 관련 티켓은 커뮤니티에 열려 있지만 PR은 닫힌 상태다."
---

Kafka 클러스터가 버전 업되면서 rack awareness를 통한 네트워크 비용 절약이 가능해졌다. 같은 AZ에 있는 컨슈머가 같은 AZ의 파티션 리플리카를 읽으면 cross-AZ 트래픽을 줄일 수 있다.

EMR 기반 Spark 잡에도 이걸 적용하고 싶었다. 각 AZ에 배포된 executor가 같은 AZ의 Kafka 파티션을 담당하게 하면 된다. 결론부터 말하면 Spark가 이걸 지원하지 않는다.

---

## Rack Awareness란

Kafka의 rack awareness는 컨슈머가 자신의 위치(rack 또는 AZ)를 브로커에 알려주면, 브로커가 가장 가까운 리플리카에서 데이터를 제공하는 기능이다. KIP-881로 컨슈머 파티션 할당 시에도 rack 정보를 반영하도록 개선됐다.

설정은 간단하다. 컨슈머에 `client.rack` 속성을 지정하면 된다. 예를 들어 `ap-northeast-2a`를 지정하면 해당 AZ의 리플리카에서 읽게 된다.

문제는 Spark에서 이 설정을 어떻게 주입하느냐다.

---

## 코드 분석: 어디에 주입해야 하나

먼저 코드베이스에서 `client.rack` 설정을 주입할 위치를 파악했다.

Kafka 토픽 읽기 흐름은 이랬다: 개별 잡 → `LogStoreProcessor.getKafkaLogDataFrame()` → `SparkSessionManager.rowKafkaDF()`. `rowKafkaDF`에서 Kafka 토픽을 읽어 데이터프레임으로 변환하는 부분에 `client.rack`을 넣으면 될 것으로 판단했다.

쓰기 쪽도 확인했다. Kafka 토픽에 쓸 때는 rack awareness가 일반적으로 의미 없지만, 브로커 설정에 따라 파티션 리더 재선출에 영향을 줄 수 있다고 해서 쓰기 로직에도 주입하려 했다.

---

## AZ 정보 추출 방법 검토

`client.rack`에 넣을 현재 AZ 정보를 어떻게 가져올지 두 가지 방안을 검토했다.

### 1안: Kubernetes Downward API

Kubernetes의 Downward API를 통해 노드의 토폴로지 라벨 정보를 컨테이너에 주입하는 방식이다.

문제가 있었다.

- AZ 정보는 팟 라벨이 아닌 노드 라벨에만 있다
- 현재 Downward API는 팟 라벨만 주입 가능하고 노드 라벨은 주입할 수 없다
- KEP-4742에서 노드 토폴로지 라벨을 Downward API로 노출하는 기능이 제안됐고 알파 피쳐로 릴리즈됐지만, 운영 중인 EKS 버전에서는 사용 불가능하다

노드명을 `spec.nodeName`에서 추출하고 K8s API로 노드 라벨을 조회하는 우회 방법도 있지만, initContainer 추가나 코드 수정이 필요하고 EMR-on-EKS에서는 pod template 파일을 S3에 올려서 관리해야 한다. 효용 대비 공수가 너무 컸다.

### 2안: AWS IMDS

훨씬 간단한 방법이 있었다. AWS 인스턴스 메타데이터 서비스(IMDS)를 통해 현재 AZ 정보를 바로 가져올 수 있다.

```bash
curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone
# ap-northeast-2b
```

EC2 인스턴스와 그 위에 배포된 컨테이너에서 고정 IP(`169.254.169.254`)로 접근 가능하다. Kubernetes Downward API 같은 추가 구성이 필요 없다. EMR-on-EC2 환경의 Spark 잡에도 동일하게 적용된다.

이 방식으로 AZ 정보를 추출하는 유틸리티 클래스를 구현했다.

---

## 그런데 Spark가 지원하지 않는다

AZ 정보를 추출하는 것까지는 해결했다. 문제는 그다음이었다.

추출한 AZ 정보를 `SparkSessionManager.rowKafkaDF()`에서 `client.rack` 형태로 Kafka 세션에 주입하려 했다. Spark의 Kafka 연동 공식 가이드를 확인했는데 rack awareness 관련 언급이 없었다.

커뮤니티를 추가로 검색해보니 현재 Spark는 Kafka 연동 시 rack awareness를 지원하지 않는다는 걸 확인했다. `client.rack`을 설정하는 것만으로는 안 된다. Spark 드라이버가 executor에 Kafka 파티션을 할당할 때 rack 정보를 고려하는 로직이 Spark 자체 코드에 추가되어야 한다.

관련 Jira 티켓(SPARK-46798)이 열려 있고, PR도 제출됐었다. 하지만 cloud vendor specific한 로직이라는 이유로 PR이 닫혀버렸다. 리뷰어들은 이런 기능이 들어가려면 정식 SPIP(Spark Improvement Proposal)가 필요하고 cloud-agnostic한 설계가 되어야 한다고 지적했다.

---

## 현재 상태와 향후 방향

Spark 커뮤니티에서 이 기능이 구현되기 전까지는 Spark Structured Streaming에서 Kafka rack awareness를 사용할 수 없다. SPARK-46798 티켓은 열려 있지만 활발하게 진행되고 있지는 않다.

한편 Kubernetes 쪽에서는 KEP-4742가 진행되고 있다. 노드 토폴로지 라벨을 팟에 자동으로 복사해주는 기능이다. EKS에서 이 기능이 사용 가능해지면 AZ 정보 추출이 더 깔끔해지지만, Spark 쪽 지원이 없으면 의미가 없다.

---

## 배운 것

**컨슈머에 `client.rack`을 설정하는 것과 파티션 할당 시 rack을 고려하는 것은 별개다.** Kafka 자체는 rack-aware 파티션 할당을 지원하지만, Spark의 Kafka 연동 레이어에서 이 정보를 활용하는 로직이 빠져 있다.

**IMDS는 AWS 환경에서 인스턴스 메타데이터를 가져오는 가장 간단한 방법이다.** Kubernetes Downward API의 한계를 우회할 수 있다. EKS든 EMR-on-EC2든 동일하게 동작한다.

**오픈소스 커뮤니티의 방향을 미리 확인하자.** 구현을 시작하기 전에 Spark 커뮤니티의 기존 논의를 먼저 확인했으면 불필요한 작업을 줄일 수 있었다.

**참고 자료:**
- [SPARK-46798: Kafka custom partition location assignment (rack awareness)](https://issues.apache.org/jira/browse/SPARK-46798)
- [SPARK-46798 PR (closed)](https://github.com/apache/spark/pull/46863)
- [Spark Kafka Rack Aware Consumer - Apache Mailing List](https://lists.apache.org/thread/t0m0hy3tl8t6kyy7vvsshvckvznsk5zs)
- [Structured Streaming + Kafka Integration Guide](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html)
- [KEP-4742: Expose Node Topology Labels via Downward API](https://github.com/kubernetes/enhancements/issues/4742)
- [K8s Issue: Exposing node labels via Downward API](https://github.com/kubernetes/kubernetes/issues/40610)
- [KIP-881: Rack-aware Partition Assignment for Kafka Consumers](https://cwiki.apache.org/confluence/display/KAFKA/KIP-881:+Rack-aware+Partition+Assignment+for+Kafka+Consumers)
