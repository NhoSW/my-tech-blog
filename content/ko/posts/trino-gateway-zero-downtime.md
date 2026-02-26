---
title: "Trino Gateway 도입기: 제로 다운타임 배포와 멀티 클러스터 라우팅"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [trino, trino-gateway, kubernetes, blue-green, routing, karpenter]
showTableOfContents: true
summary: "Trino는 코디네이터 HA를 지원하지 않는다. 코디네이터를 재배포하면 다운타임이 생긴다. Trino Gateway를 도입해서 Blue/Green 배포로 제로 다운타임을 달성하고 BI/OLAP 클러스터를 헤더 기반으로 라우팅한 과정을 정리했다."
---

Trino 코디네이터는 단일 장애점이다. HA를 지원하지 않아서 코디네이터가 내려가면 클러스터 전체가 쿼리를 받지 못한다. 프로덕션에서 이게 문제가 되는 순간은 크게 두 가지다.

하나는 **노드 교체**. Kubernetes 환경에서 Karpenter로 노드를 관리하는데 장기간 실행 중인 노드는 네트워크 이슈가 생길 수 있다. `ttlSecondsUntilExpired` 옵션으로 주기적으로 교체하고 싶지만 코디네이터가 올라간 노드가 교체되는 시점을 제어하기 어렵다.

다른 하나는 **배포**. 코디네이터 팟을 재배포하면 새 팟이 뜰 때까지 다운타임이 발생한다. 새벽에 CronJob으로 돌리는 방법도 있지만 그 시간대에 실행 중인 배치 쿼리가 실패할 수 있다.

Lyft에서 이 문제를 presto-gateway로 해결했다는 걸 Deview 발표에서 알게 됐다. 여러 Trino 클러스터 앞에 게이트웨이를 두면 Blue/Green 배포가 가능해지고 다운타임 없이 코디네이터를 교체할 수 있다.

---

## Trino Gateway란

Trino Gateway는 여러 Trino 클러스터 앞에 놓는 로드 밸런서이자 라우팅 프록시다. 원래 Lyft가 presto-gateway라는 이름으로 개발했고 지금은 trinodb 조직 아래에서 trino-gateway로 활발하게 유지보수되고 있다.

핵심 기능은 세 가지다.

- **멀티 클러스터 라우팅**: 쿼리를 조건에 따라 다른 클러스터 그룹으로 보낼 수 있다
- **백엔드 헬스 체크**: 뒤에 있는 클러스터가 정상인지 주기적으로 확인하고 장애 클러스터를 자동으로 제외한다
- **큐 체크**: 각 백엔드의 현재 쿼리 수를 기준으로 부하를 분산한다

클라이언트 입장에서는 게이트웨이 주소 하나만 알면 된다. 뒤에 클러스터가 몇 개인지, 어느 클러스터가 살아있는지 신경 쓸 필요가 없다.

---

## 아키텍처

최종 목표 구성은 이렇다.

```
Trino Gateway
  ├── BI Cluster Group
  │     ├── BI Cluster 1 (Coordinator: B존)
  │     └── BI Cluster 2 (Coordinator: C존)
  │
  └── OLAP Cluster Group
        ├── OLAP Cluster 1 (Coordinator: A존)
        └── OLAP Cluster 2 (Coordinator: B존)
```

코디네이터를 서로 다른 가용 영역(AZ)에 분산시킨 건 의도적이다. 과거에 특정 AZ의 노드 프로비저닝 장애로 코디네이터가 뜨지 못한 적이 있어서다. AZ를 분산하면 한 AZ가 장애를 겪어도 다른 AZ의 클러스터가 쿼리를 받을 수 있다.

### 운영 방식

리소스 낭비를 줄이기 위해 각 클러스터 그룹에서 동시에 하나의 클러스터만 활성화한다. 롤링 배포 시점에만 일시적으로 두 클러스터가 동시에 뜬다.

```
평소:     Gateway → Cluster 1 (활성)     Cluster 2 (비활성)
롤링 중:  Gateway → Cluster 1 (활성) + Cluster 2 (부팅 중)
완료 후:  Gateway → Cluster 2 (활성)     Cluster 1 (비활성)
```

게이트웨이가 헬스 체크로 새 클러스터가 준비됐음을 확인하면 트래픽을 넘기고 기존 클러스터의 실행 중인 쿼리가 끝나길 기다린 뒤 비활성화한다.

---

## 헤더 기반 라우팅 룰

쿼리 소스에 따라 적절한 클러스터 그룹으로 라우팅한다. Trino 클라이언트가 보내는 HTTP 헤더를 기준으로 분기한다.

```yaml
# Superset → BI 클러스터
- name: "superset"
  condition: >
    request.getHeader("X-Trino-Source") == "Apache Superset"
    && request.getHeader("X-Trino-Client-Tags") == null
  actions:
    - "result.put(\"routingGroup\", \"bi\")"

# Querybook → OLAP 클러스터
- name: "querybook"
  condition: >
    request.getHeader("X-Trino-Source") == "trino-python-client"
    && request.getHeader("X-Trino-Client-Tags") == null
  actions:
    - "result.put(\"routingGroup\", \"olap\")"

# Zeppelin → OLAP 클러스터
- name: "zeppelin"
  condition: >
    request.getHeader("X-Trino-Source") ~= "^zeppelin-.+"
  actions:
    - "result.put(\"routingGroup\", \"olap\")"
```

`X-Trino-Source` 헤더는 Trino 클라이언트가 자동으로 붙여준다. Superset은 `Apache Superset`을 보내고 Querybook은 `trino-python-client`를 보낸다. Zeppelin은 `zeppelin-`으로 시작하는 소스명을 쓴다.

`X-Trino-Client-Tags`가 null인 조건을 추가한 건 특정 태그가 붙은 쿼리를 별도 처리하기 위한 여지를 남겨둔 것이다.

---

## 백엔드 헬스/큐 체크 정상화

게이트웨이를 배포하고 나서 백엔드 헬스 체크와 큐 체크에 문제가 있음을 발견했다.

### 헬스 체크 문제

게이트웨이가 백엔드 클러스터의 상태를 확인할 때 Trino의 `/v1/info` 엔드포인트를 찌른다. 그런데 코디네이터가 시작 중인 상태에서도 이 엔드포인트가 200을 반환하는 경우가 있었다. 게이트웨이는 클러스터가 준비됐다고 판단하고 쿼리를 보내는데 실제로는 아직 쿼리를 처리할 수 없는 상태였다.

### 큐 체크 문제

각 백엔드의 현재 실행 중인 쿼리 수와 대기 중인 쿼리 수를 확인해서 부하를 분산하는 로직이 제대로 동작하지 않고 있었다. 특정 클러스터에 쿼리가 쏠리는 현상이 발생했다.

### 수정 내용

게이트웨이 코드와 Trino 클러스터 설정 양쪽을 수정했다.

- 게이트웨이 쪽: 헬스 체크 로직을 보강해서 코디네이터가 완전히 준비된 상태인지 확인하도록 수정. 큐 체크에서 쿼리 수 기반 부하 분산이 정확하게 동작하도록 수정
- Trino 클러스터 쪽: 차트 템플릿과 설정을 변경해서 게이트웨이와의 연동이 올바르게 동작하도록 조정

---

## 매일 돌리는 클러스터 롤링 배치

게이트웨이 도입의 가장 큰 이점은 **주간 시간대에도 다운타임 없이 클러스터를 교체**할 수 있다는 점이다.

Airflow DAG으로 매일 클러스터 롤링 배치를 구성했다. 동작 방식은 이렇다.

1. 비활성 클러스터의 새 코디네이터와 워커를 띄운다
2. 게이트웨이 헬스 체크가 새 클러스터를 정상으로 판단할 때까지 대기한다
3. 새 클러스터가 준비되면 게이트웨이가 새 쿼리를 새 클러스터로 라우팅한다
4. 기존 클러스터에서 실행 중인 쿼리가 완료될 때까지 기다린다
5. 기존 클러스터를 비활성화한다

이렇게 하면 장기 실행 노드에서 생기는 네트워크 이슈를 예방하면서도 실행 중인 쿼리에 영향을 주지 않는다.

---

## 도입 과정

단계적으로 진행했다.

### 1단계. 테스트 환경 PoC

테스트 환경에서 게이트웨이를 먼저 구축하고 기본적인 라우팅과 헬스 체크 동작을 확인했다.

### 2단계. 스테이지 환경 구축

프로덕션 환경(D01)에 스테이지를 구축하고 Superset, Querybook 등 실제 클라이언트 연동을 검증했다. 이 단계에서 헬스 체크와 큐 체크 문제를 발견하고 수정했다.

### 3단계. 프로덕션 적용

Superset, Querybook, Zeppelin에 우선 적용했다. 게이트웨이 주소로 엔드포인트를 전환하고 라우팅이 정상 동작하는지 모니터링했다.

### 4단계. 롤링 배치 운영

매일 클러스터 롤링 배치를 Airflow DAG으로 운영하기 시작했다. Beta 환경에서 정상 동작을 확인한 뒤 프로덕션에 적용했다.

---

## 마치며

Trino Gateway 도입으로 해결한 문제를 정리하면 이렇다.

**제로 다운타임 배포.** 코디네이터 재배포 때 다운타임이 사라졌다. Blue/Green 방식으로 새 클러스터가 준비된 후 트래픽을 넘기니까 쿼리 하나 떨어뜨리지 않고 교체할 수 있다.

**AZ 장애 내성.** 코디네이터를 서로 다른 AZ에 분산시켜서 단일 AZ 장애에도 쿼리를 처리할 수 있게 됐다.

**워크로드 분리.** BI 쿼리는 BI 클러스터로, OLAP 쿼리는 OLAP 클러스터로 자동 라우팅된다. 클라이언트는 게이트웨이 주소 하나만 알면 된다.

**노드 신선도 유지.** 매일 클러스터를 롤링해서 장기 실행 노드에서 발생하는 네트워크 이슈를 예방한다.

헬스 체크와 큐 체크 로직을 수정하는 데 시간이 좀 들었지만 그 과정을 거치고 나니 게이트웨이가 안정적으로 동작한다. Trino를 프로덕션에서 운영한다면 게이트웨이는 선택이 아니라 필수에 가깝다.

**참고 자료:**
- [Trino Gateway (trinodb)](https://github.com/trinodb/trino-gateway)
- [Presto Infrastructure at Lyft](https://eng.lyft.com/presto-infrastructure-at-lyft-b10adb9db01)
- [Trino Open Source Infrastructure Upgrading at Lyft](https://eng.lyft.com/trino-open-source-infrastructure-upgrading-at-lyft-83f26b099fa)
- [Presto Gateway (Lyft, legacy)](https://github.com/lyft/presto-gateway)
