---
title: "AWS EC2 인스턴스 아키텍처 비교: ARM Graviton4 vs AMD Turin — 가장 빠른 인스턴스가 최선인가"
date: 2026-03-09
draft: false
categories: [Data Engineering]
tags: [aws, ec2, graviton, arm, amd, spot-instance, trino, spark, karpenter, cost-optimization]
showTableOfContents: true
summary: "2026년 벤치마크에서 AMD EPYC Turin(C8a)이 단일/멀티스레드 모두 압도적 1위를 기록했다. 현재 Graviton(ARM) 기반으로 운영 중인 데이터 플랫폼 인프라를 전환해야 할까? vCPU와 물리 Core의 차이, Spot 가격 대비 Core 확보 효율, 워크로드별 특성을 분석한 결과 — 전면 교체보다 혼합 전략이 합리적이라는 결론에 도달했다."
---

데이터플랫폼팀은 Trino, Spark, Flink, StarRocks 등 대규모 분산 처리 워크로드를 AWS 서울 리전에서 운영하고 있다. 주요 연산 인스턴스로 Graviton(ARM) 기반 EC2를 채택해 x86 대비 10~20% 낮은 비용에 동급 이상의 성능을 확보해왔다.

2026년 2월 공개된 클라우드 VM 벤치마크에서 AMD EPYC Turin 기반 AWS C8a 인스턴스가 단일 스레드와 멀티스레드 성능 모두 압도적 1위를 기록했다. 이 결과를 보고 질문이 생겼다. 현재 ARM 기반 인프라를 C8a로 전환해야 하는가?

결론부터 말하면, 전면 교체는 비용 효율 측면에서 최선이 아니다. 이유를 정리한다.

---

## vCPU ≠ 물리 Core: 가장 중요하지만 자주 간과되는 차이

인스턴스 비교에서 가장 먼저 이해해야 하는 개념이 vCPU와 물리 Core의 차이다. 이걸 모르면 같은 32 vCPU 인스턴스를 사면서 실제 연산 능력이 2배 차이 나는 상황이 생긴다.

Intel과 AMD 인스턴스 대부분은 SMT(Simultaneous Multi-Threading)를 활성화한다. 하나의 물리 Core를 2개의 논리 스레드(vCPU)로 분할해 노출하는 기술이다. CPU가 메모리 접근 등으로 대기하는 유휴 시간을 다른 스레드가 활용할 수 있어 처리율이 올라가지만, 두 스레드가 같은 물리 자원을 공유하므로 CPU-bound 워크로드에서 성능이 2배로 늘어나지는 않는다.

| 인스턴스 | 아키텍처 | SMT | 32 vCPU 기준 물리 Core |
|---------|---------|-----|----------------------|
| C8g (Graviton4) | ARM | 없음 | **32 Core** |
| C8a (AMD Turin) | x86 | **비활성화** | **32 Core** |
| C7i (Intel Sapphire Rapids) | x86 | 활성화 | **16 Core** |
| C7g (Graviton3) | ARM | 없음 | **32 Core** |

C8a는 AWS가 SMT를 의도적으로 비활성화해서 vCPU 1개 = 물리 Core 1개를 보장한다. Graviton은 원래 SMT가 없다. 반면 C7i(Intel)는 32 vCPU를 사면 실제 물리 Core가 16개뿐이다.

Spark executor는 `--executor-cores`로 vCPU를 할당받는다. HT 인스턴스에서 4 vCPU는 실제 2 물리 Core이므로 병렬 연산 처리가 절반에 불과하다. Trino worker의 task parallelism도 물리 Core 수에 비례한다. shuffle, sort, hash join, 집계 등 CPU-bound 연산에서 물리 Core 수가 throughput을 직접 결정한다.

---

## 벤치마크: Turin이 압도적이다, 그런데

2026년 벤치마크 결과에서 AMD Turin(C8a)이 거의 모든 항목에서 1위를 차지했다. Graviton3를 100으로 정규화한 상대 성능은 다음과 같다.

| 인스턴스 | 단일 스레드 | 멀티스레드 |
|---------|-----------|----------|
| C8a (AMD Turin) | **145** | **160** |
| C8g (Graviton4) | 125 | 130 |
| C7i (Intel Sapphire Rapids) | 110 | 105 |
| C7g (Graviton3) | 100 | 100 |

Turin이 Graviton4보다 단일 스레드 16%, 멀티스레드 23% 빠르다. Graviton3 대비로는 45~60% 차이가 난다. 수치만 보면 C8a 전환이 당연해 보인다.

### 워크로드별 특이점

모든 벤치마크에서 Turin이 압도적인 것은 아니다.

**7-zip 압축 해제**: Graviton4와 Graviton3가 Turin을 앞서는 경우가 있다. Iceberg 파일 스캔에서 Parquet 해제가 많은 경우 ARM이 유리할 수 있다.

**OpenSSL RSA4096 (AVX512)**: Turin > Genoa > Intel 순서. ARM은 AVX512를 지원하지 않아 하위권이다. 단, Trino와 Spark는 일반적으로 AVX512 의존도가 낮다. StarRocks BE의 벡터 집계 연산은 예외적으로 AVX512 혜택을 받을 수 있다.

**NGINX 단일 스레드**: C8a가 2위 대비 거의 2배, 3위 대비 3배 성능을 보인다. Trino coordinator처럼 단일 스레드 응답 시간이 latency에 직결되는 서비스에서 의미 있는 지표다.

---

## 가격: 성능이 아니라 Core당 비용이 핵심이다

벤치마크에서 C8a가 가장 빠르다는 것은 확인했다. 문제는 가격이다.

### 8xlarge (32 vCPU) Spot 가격 비교

| 인스턴스 | Spot 월간 비용 | 물리 Core | Core당 Spot 단가 |
|---------|-------------|----------|----------------|
| C7g (Graviton3) | ~$220 | 32 | **$6.9** |
| C8g (Graviton4) | ~$237 | 32 | **$7.4** |
| C8a (AMD Turin)* | ~$340 | 32 | **$10.6** |
| C7i (Intel) | ~$290 | 16 | **$18.1** |

*C8a 서울 리전 가격은 us-east-1 실측에 서울 프리미엄(~12%)을 적용한 추정값.

C8a는 Core당 성능이 뛰어나지만 Spot 가격이 Graviton4보다 43% 비싸다. Core당 단가 차이가 성능 차이보다 크다.

### 동일 예산($500/월 Spot)으로 확보 가능한 Core 수

| 인스턴스 | 확보 Core 수 | 상대 성능/Core | 유효 처리력 |
|---------|-------------|--------------|-----------|
| C7g | ~72 | 1.0x | 72 |
| C8g | ~67 | 1.3x | **87** |
| C8a* | ~47 | 1.6x | 75 |
| C7i | ~28 | 1.05x | 29 |

Graviton4(C8g)가 동일 예산에서 가장 높은 유효 처리력을 보인다. C8a는 Core당 성능이 뛰어나지만 확보 가능한 Core 수가 적어서, 유효 처리력으로 환산하면 Graviton4에 밀린다. Intel은 HT 때문에 Core 효율이 최하위다.

---

## 워크로드별 분석

### Spark 배치/ETL

Spark 배치는 대규모 병렬 처리가 핵심이다. `executor-cores`로 task parallelism을 늘리므로 물리 Core가 많을수록 throughput이 선형에 가깝게 증가한다. 단일 스레드 클럭 속도보다 Core 수가 우선이다.

EKS Spot 환경에서는 인터럽션 시 executor 재시도로 처리된다. Celeborn RSS 등을 활용하면 더욱 안정적이다. 고성능 인스턴스 소수보다 저렴한 인스턴스 다수를 병렬 운영하는 전략이 유리하다. Graviton4가 Spot 가격 대비 Core 확보 효율이 가장 뛰어나다.

**추천: C8g (Graviton4) Spot**

### Trino 대화형 쿼리

Trino는 Spark과 성격이 다르다. 쿼리 응답 시간(latency)이 중요하고, coordinator가 단일 스레드로 쿼리 플랜을 처리하는 단계가 병목이 될 수 있다.

C8a의 압도적 단일 스레드 성능은 coordinator의 쿼리 계획/파싱 속도를 줄이는 데 직접적으로 기여한다. Worker 노드는 병렬 스캔, 집계, 조인이 중심이므로 Core 수가 충분히 많아야 한다.

**추천: Coordinator는 C8a (On-Demand), Worker는 C8g Spot**

### Iceberg 관리 작업

Compaction, OPTIMIZE, REWRITE MANIFESTS 등은 파일 I/O와 병렬 정렬이 핵심이다. 220개 이상의 Iceberg 테이블을 관리하는 환경에서 Core 수 확보가 작업 완료 시간을 단축한다. Spot 인터럽션에 재시도로 대응 가능하므로 가성비가 높은 Graviton 계열이 적합하다.

**추천: C8g (Graviton4) Spot**

### StarRocks BE

StarRocks Backend는 벡터 집계 연산을 수행하며 AVX512 명령어 집합을 활용할 수 있다. Turin은 AMD 최신 AVX512 구현체로 벤치마크에서 Intel을 압도했다. Dedicated 클러스터로 운영하는 경우 C8a 전환이 가장 명확한 효과를 낼 수 있는 영역이다.

**추천: C8a (On-Demand 또는 Reserved)**

---

## 전환 판단 기준

C8a가 서울 리전에 출시되더라도 무조건 전환이 정답은 아니다.

### C8a 전환을 고려해야 하는 경우

- Trino 쿼리 응답시간이 SLA를 지속 초과하고 CPU가 명확한 병목인 경우
- 단일 쿼리 처리 시간(특히 coordinator 플랜 생성)이 Core 수 증가로 해결되지 않는 경우
- StarRocks BE의 집계 연산이 CPU bound이며 AVX512 활용이 중요한 경우

### Graviton 유지가 합리적인 경우

- Spot 기반 Spark 배치가 정상 운영 중이며 Core 부족이 병목이 아닌 경우
- 동일 예산에서 Core 수 최대화가 전략적 목표인 경우
- ARM 네이티브 빌드(Docker 이미지, 라이브러리)가 완비되어 마이그레이션 비용이 없는 경우
- C8a Spot 가용성이 서울 리전에서 낮을 경우 (신규 인스턴스 출시 초기에 흔함)

---

## 전환 시 실무 고려사항

### ARM → x86(C8a) 전환

- **Docker 이미지**: ARM64 → AMD64 재빌드 필요. ECR 멀티 아키텍처 이미지 관리 권장
- **JVM 플래그**: Graviton에서 사용하던 `-XX:+UseNUMA`, `-XX:+UseTransparentHugePages` 등 ARM 최적화 플래그 검토
- **Native 라이브러리**: Arrow, Snappy, LZ4 등 JNI 기반 라이브러리의 AMD64 바이너리 확인
- **Spot 가용성**: 신규 인스턴스는 초기에 Spot Pool이 작아 인터럽션이 잦을 수 있음. 출시 후 3~6개월 관찰 권장

### Graviton3 → Graviton4 업그레이드

- 동일 ARM 아키텍처(arm64). 이미지, 라이브러리, 설정 변경 불필요
- Spot 가격 약 8% 증가로 약 30% 성능 향상
- EKS nodegroup AMI만 업데이트하면 적용 가능한 가장 리스크 낮은 업그레이드 경로

---

## 결론

2026년 벤치마크는 AMD Turin의 압도적 성능을 확인시켜 주었다. 하지만 "가장 빠른 인스턴스 = 가장 좋은 선택"은 아니다.

Trino와 Spark 기반 분산 데이터 플랫폼에서는 세 가지를 종합적으로 봐야 한다.

1. **물리 Core 수**: Spot 병렬 워크로드에서 Core 수가 throughput을 결정한다
2. **Spot 가격 가성비**: 동일 예산으로 더 많은 Core를 확보할 수 있다면 분산 처리가 유리하다
3. **워크로드 특성**: 단일 스레드 latency인지, 병렬 throughput인지 구분해야 한다

Graviton4 기반 ARM 전략을 핵심으로 유지하면서, C8a의 압도적 단일 스레드 성능을 latency-critical 컴포넌트(Trino coordinator, StarRocks BE)에 선택적으로 활용하는 혼합 전략을 권장한다.

**참고 자료:**
- [Cloud VM benchmarks 2026: performance / price](https://dev.to/dkechag/cloud-vm-benchmarks-2026-performance-price-p0p)
- [AWS EC2 C8g Instance Types](https://aws.amazon.com/ec2/instance-types/c8g/)
- [AWS EC2 Spot Pricing](https://aws.amazon.com/ec2/spot/pricing/)
- [AMD EPYC 9004 Series (Turin)](https://www.amd.com/en/products/processors/server/epyc/9005-series.html)
