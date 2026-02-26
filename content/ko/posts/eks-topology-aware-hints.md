---
title: "EKS Topology Aware Hint 적용 검토: 우리 클러스터에서는 의미 없었던 이유"
date: 2026-02-27
draft: false
categories: [Data Engineering]
tags: [kubernetes, eks, networking, aws, cost-optimization]
showTableOfContents: true
summary: "EKS cross-AZ 통신 비용 절감을 위해 Topology Aware Hint 적용을 검토했다. EndpointSlice에 힌트는 정상적으로 붙었지만, 실제 효과는 없었다. AWS Load Balancer Controller의 IP 타깃 모드가 kube-proxy를 우회하고, 주요 내부 워크로드(Spark, Trino, Airflow)가 모두 싱글 존 또는 stateful 통신이라 힌트가 참조되는 경로 자체가 존재하지 않았다."
---

EKS 클러스터의 cross-AZ 통신 비용이 무시할 수 없는 수준이다. AWS에서는 동일 리전 내라도 AZ 간 트래픽에 GB당 $0.01의 요금을 부과한다. 워크로드가 여러 AZ에 분산되어 있으면 이 비용이 빠르게 쌓인다.

Kubernetes 1.23부터 제공되는 Topology Aware Hint(이하 TAH)는 이 문제를 완화하는 기능이다. 서비스의 EndpointSlice에 존(AZ) 정보를 힌트로 붙여서, kube-proxy가 트래픽을 라우팅할 때 같은 AZ의 엔드포인트를 우선적으로 선택하도록 유도한다.

AWS 공식 블로그에서도 EKS 환경에서 TAH를 활용한 cross-AZ 비용 절감 사례를 소개하고 있었다. 우리 클러스터에도 적용할 수 있을까? 테스트를 진행했다.

결론부터 말하면, 우리 클러스터 환경에서는 TAH가 의미 없었다. 이유를 정리한다.

---

## 테스트 환경 구성

AWS 블로그의 가이드를 참고해서 간단한 에코 서버를 멀티 AZ에 배포하고, TAH 활성화 후 동작을 확인했다.

서비스에 `service.kubernetes.io/topology-aware-hints: auto` 어노테이션을 추가하면, EndpointSlice 컨트롤러가 각 엔드포인트에 `hints.forZones` 필드를 붙여준다. 이 힌트는 "이 엔드포인트는 이 AZ의 트래픽을 처리하는 데 적합하다"는 의미다.

테스트 결과, EndpointSlice 컨트롤러가 개별 엔드포인트에 힌트를 정상적으로 부여하는 것을 확인했다. 기능 자체는 의도한 대로 동작하고 있었다.

하지만 더미 파드의 셸에서 서비스에 접근해보면, 힌트대로 같은 AZ의 엔드포인트로만 라우팅되지 않았다. TAH가 실제 트래픽 라우팅에 영향을 주려면 몇 가지 전제 조건이 충족되어야 하는데, 우리 환경에서는 그 조건이 맞지 않았다.

---

## 외부 트래픽: AWS Load Balancer Controller가 kube-proxy를 우회한다

우리 클러스터의 외부 트래픽 진입 경로부터 살펴보자.

현재 ingress controller로 nginx-ingress가 아닌 AWS Load Balancer Controller를 사용하고 있다. AWS EKS 베스트 프랙티스에서 권고하는 대로 IP 타깃 유형으로 ingress 리소스를 생성한다.

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

IP 타깃 유형을 사용하면 ALB가 파드의 IP를 직접 타깃으로 등록한다. 트래픽이 노드의 kube-proxy를 거치지 않고 파드에 직접 도달한다. 이것이 핵심이다.

TAH는 kube-proxy가 트래픽을 라우팅할 때 참조하는 힌트다. kube-proxy가 iptables 또는 IPVS 규칙을 설정할 때 힌트 정보를 반영해서 같은 AZ의 엔드포인트를 우선 선택한다. 그런데 IP 타깃 유형 ingress에서는 kube-proxy를 아예 거치지 않으니, TAH 힌트가 참조될 경로 자체가 없다.

nginx-ingress controller를 사용하면 상황이 다르다. nginx 파드와 업스트림 파드 사이의 트래픽이 kube-proxy를 거치기 때문에 TAH 힌트가 유효하다. 하지만 우리 클러스터에서는 과거에 사용하던 nginx-ingress를 제거하고 AWS Load Balancer Controller만을 사용하고 있다.

외부 트래픽에 대해서는 TAH가 적용 불가능하다.

---

## 내부 트래픽: 적용 요건에 맞는 워크로드가 없다

그렇다면 클러스터 내부 서비스 간 통신은 어떨까? 파드끼리 Service를 통해 통신하면 kube-proxy를 거치므로 TAH가 유효할 수 있다.

하지만 TAH가 의미 있으려면 몇 가지 제약 조건이 충족되어야 한다.

### 제약 조건

**다대다 멀티 AZ 배포:** 클라이언트와 서버 양쪽이 다수의 파드로 여러 AZ에 분산되어 있어야 한다.

**AZ별 균등한 엔드포인트 분포:** 서버 측이 모든 가용 영역(A, B, C)에 걸쳐 배포되어 있어야 하고, 각 AZ별 노드의 총 코어 수 대비 충분한 수의 엔드포인트가 있어야 한다. EndpointSlice 컨트롤러는 AZ별 코어 비율에 따라 힌트를 배분하는데, 특정 AZ에 엔드포인트가 부족하면 힌트가 제대로 동작하지 않는다.

**클라이언트의 AZ 분포:** 클라이언트 쪽이 특정 존에 편중되어 있으면 힌트가 제대로 동작하지 않는다. 또한 특정 존의 노드 부하 메트릭이 전체 노드 부하를 대표하지 못하게 되어, HPA 같은 오토스케일러와 호환성 문제가 생길 수 있다.

**Stateless 통신:** 클라이언트가 서버 측 서비스의 어떤 엔드포인트 파드로 접근해도 문제없는 stateless 통신이어야 한다.

### 우리 워크로드 검토

이 제약 조건을 우리 클러스터의 주요 내부 통신에 대입해봤다.

| 워크로드 | 통신 패턴 | TAH 적용 가능성 |
|---------|----------|---------------|
| Spark driver ↔ executor | 싱글 존 배포, stateful | 불가 |
| Trino coordinator ↔ worker | 싱글 존 배포, stateful | 불가 |
| Trino Gateway ↔ Trino 클러스터 | 개별 클러스터가 싱글 존, stateful (특정 쿼리는 해당 백엔드로만 라우팅) | 불가 |
| Airflow scheduler ↔ worker ↔ metaDB(RDS) ↔ Redis(ElastiCache) | stateful | 불가 |

Spark와 Trino는 의도적으로 싱글 존에 배포하고 있다. driver-executor, coordinator-worker 간 통신이 매우 빈번하고 대용량이기 때문에, cross-AZ 비용을 원천 차단하기 위해 동일 AZ에 모든 컴포넌트를 배치한다. 이미 싱글 존이므로 TAH가 해결하려는 문제 자체가 존재하지 않는다.

Trino Gateway와 백엔드 Trino 클러스터 간 통신은 stateful이다. 게이트웨이가 특정 쿼리를 특정 백엔드 클러스터로 라우팅하면, 해당 쿼리의 모든 후속 통신은 그 클러스터로만 간다. 이런 통신 패턴에서는 "가까운 엔드포인트를 선택한다"는 TAH의 개념이 적용될 수 없다.

Airflow 역시 scheduler, worker, metaDB, Redis 간의 통신이 stateful하다. scheduler가 특정 worker에 태스크를 배정하면 해당 worker와의 통신이 유지되고, metaDB와 Redis는 고정된 엔드포인트다.

결론적으로, TAH 적용 요건에 부합하는 내부 통신 사례가 없었다.

---

## 적용 가치가 있는 경우

우리 클러스터에서는 의미가 없었지만, TAH가 유효한 환경은 분명히 있다.

- nginx-ingress controller를 사용하는 클러스터에서 nginx 파드와 업스트림 파드 간 트래픽
- 전형적인 마이크로서비스 아키텍처에서 멀티 AZ에 배포된 stateless 서비스 간 통신
- 추천 시스템의 API 서버처럼, 클라이언트와 서버 양쪽이 멀티 AZ에 다수의 파드로 분산된 경우

실제로 이 검토 결과를 사내 추천팀에 공유했더니, 그쪽 워크로드에서는 적용 가치가 있을 수 있다는 피드백을 받았다. 워크로드 특성에 따라 판단이 달라진다.

---

## 비고

몇 가지 추가 참고 사항이다.

**어노테이션 키 변경:** Kubernetes v1.27부터 TAH 어노테이션 키가 `service.kubernetes.io/topology-aware-hints`에서 `service.kubernetes.io/topology-mode`로 변경됐다. 우리 EKS 버전(v1.24 기준)에는 해당되지 않았지만, 향후 버전 업그레이드 시 참고가 필요하다.

**커스텀 휴리스틱:** 라우팅 힌트에 커스텀 휴리스틱 룰을 붙이는 기능이 개발 중이다. 현재의 AZ 기반 힌트 외에 사용자 정의 라우팅 로직을 추가할 수 있게 되면 활용 가능성이 넓어질 수 있다.

---

## 배운 것

**기능이 동작하는 것과 효과가 있는 것은 다르다.** EndpointSlice에 힌트가 정상적으로 붙는다고 해서 트래픽 라우팅에 영향을 주는 건 아니다. 힌트가 참조되는 경로(kube-proxy)를 트래픽이 실제로 거치는지를 먼저 확인해야 한다.

**AWS Load Balancer Controller의 IP 타깃 모드는 kube-proxy를 완전히 우회한다.** TAH뿐 아니라, kube-proxy 기반으로 동작하는 모든 네트워크 기능(NetworkPolicy 일부 구현 포함)이 IP 타깃 모드에서는 영향을 받을 수 있다. 네트워크 레벨 기능을 적용할 때 트래픽 경로를 정확히 이해하는 것이 중요하다.

**cross-AZ 비용 절감은 네트워크 기능보다 배치 전략이 더 효과적일 수 있다.** Spark과 Trino를 싱글 존으로 배포하는 것이 cross-AZ 트래픽을 원천 차단하는 가장 확실한 방법이었다. 네트워크 레벨에서 AZ를 고려하는 것보다, 워크로드 배치에서 AZ를 통제하는 것이 더 단순하고 효과적이다.

**적용 전에 워크로드 통신 패턴을 먼저 분석하자.** TAH의 제약 조건(멀티 AZ, 균등 분포, stateless)에 부합하는 워크로드가 없으면 설정을 켜봤자 효과가 없다. 블로그나 문서의 사례가 자신의 환경에도 적용되는지 확인하는 단계를 건너뛰면 시간을 낭비하게 된다.

**참고 자료:**
- [Amazon EKS: Reduce cross-AZ traffic costs with Topology Aware Hints](https://aws.amazon.com/ko/blogs/tech/amazon-eks-reduce-cross-az-traffic-costs-with-topology-aware-hints/)
- [Kubernetes: Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/)
- [Kubernetes: Topology Aware Routing Constraints](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/#constraints)
- [EKS Best Practices: Use IP target type load balancers](https://aws.github.io/aws-eks-best-practices/networking/loadbalancing/loadbalancing/#use-ip-target-type-load-balancers)
- [Kubernetes: Custom Heuristics for Topology Aware Routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/#custom-heuristics)
