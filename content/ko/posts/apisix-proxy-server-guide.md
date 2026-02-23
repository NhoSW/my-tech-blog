---
title: "Apache APISIX로 프록시 서버 구축하기: 일 13억 건 로그의 비식별화 처리"
date: 2026-02-24
draft: false
categories: [Data Engineering]
tags: [apisix, proxy, lua, kubernetes, privacy, kafka]
showTableOfContents: true
summary: "해외 트래킹 서버로 앱 로그를 전송할 때 개인정보를 비식별화해야 했다. Fluent-bit에서 시작해 Kong을 거쳐 APISIX에 정착한 과정, 커스텀 Lua 플러그인 개발, 5.5만 TPS 부하 테스트, Kubernetes 배포까지 실전 경험을 정리했다."
---

앱 로그를 해외 트래킹 서버로 보내야 하는 상황이 생겼다. 문제는 로그 안에 회원번호, 디바이스 ID, 주문번호 같은 개인식별정보(PII)가 들어 있다는 점이다. 개인정보보호법상 이걸 그대로 국외 전송하면 안 된다.

해결책은 중간에 프록시 서버를 두는 것이다. 앱에서 보낸 로그를 프록시가 받아서 PII를 해싱한 뒤 해외 서버로 전달한다. 동시에 원본 데이터는 사내 Kafka로 보내서 광고, 추천 등 내부 데이터 프로덕트에서 쓸 수 있게 한다.

이 글은 일 평균 13억 건의 로그를 처리하는 프록시 서버를 Apache APISIX로 구축한 과정을 기록한 것이다.

---

## 왜 APISIX인가

### 처음에는 Fluent-bit을 생각했다

로그 수집이니까 Fluent-bit이 자연스러운 선택이었다. 하지만 요구사항을 정리해 보니 단순한 로그 포워딩이 아니라 **API Gateway 기능**이 필요했다.

- HTTP 요청을 받아서 request body를 조작해야 한다 (PII 해싱)
- 해외 트래킹 서버의 응답(400 에러 등)을 클라이언트에 그대로 전달해야 한다
- Kafka로 원본 데이터를 동시에 발행해야 한다

Fluent-bit은 이 중 어느 것도 깔끔하게 해결하지 못한다.

### Kong vs APISIX

둘 다 nginx 기반이고 Lua 스크립팅을 지원한다. Kong을 먼저 테스트했는데 Helm 차트 배포 단계에서 [알려진 이슈](https://github.com/Kong/charts/issues/1198)에 걸려 실패했다.

APISIX로 전환한 이유는 명확하다.

- **경량화 설계**: Kong보다 메모리 사용량이 적고 코어당 처리량이 높다
- **선언적 라우팅**: Admin API로 설정을 보내면 etcd에 저장되어 영구 보관된다
- **커스텀 플러그인**: rewrite → access → header_filter → body_filter → log 단계로 요청 처리 파이프라인이 나뉘어 있어 원하는 단계에 로직을 끼워 넣기 좋다

---

## 커스텀 플러그인 개발

### 왜 기존 플러그인으로는 안 되나

APISIX에는 `kafka-logger`라는 내장 플러그인이 있다. 요청 데이터를 Kafka로 보내주는 기능이다. 문제는 비식별화 플러그인(`encrypt-pii`)을 먼저 실행하면 kafka-logger가 **원본이 아닌 해싱된 데이터**에 접근한다는 점이다.

우리가 원하는 것은 이렇다.

```
앱 → APISIX → (1) 원본을 Kafka로 발행
                  (2) PII를 해싱
                  (3) 해싱된 데이터를 해외 서버로 전송
                  (4) 해외 서버의 응답을 앱에 반환
```

(1)과 (2)의 순서가 중요하다. 원본을 Kafka에 먼저 보내고 그 다음 해싱해야 한다. 기존 플러그인 조합으로는 이 순서를 보장할 수 없어서 **kafka-logger 기능까지 포함한 커스텀 플러그인**을 만들었다.

### Lua로 비식별화 구현

APISIX는 LuaJIT 위에서 돌아간다. `lua-resty-string` 모듈로 SHA-256 해싱을 구현했다.

```lua
local resty_sha256 = require "resty.sha256"
local str = require "resty.string"

local function hash_value(value)
    local sha256 = resty_sha256:new()
    sha256:update(value)
    local digest = sha256:final()
    return str.to_hex(digest)
end
```

비식별화 대상은 세 가지다.
- 회원번호
- 디바이스 ID
- 주문번호

request body에서 JSON을 파싱하고 대상 필드를 찾아 해싱한 뒤 다시 직렬화한다.

### Kafka 메시지 발행

같은 플러그인 안에서 `lua-resty-kafka`를 사용해 원본 데이터를 Kafka로 보낸다.

```lua
local producer = require "resty.kafka.producer"

local broker_list = {
    { host = "kafka-broker-01", port = 9092 },
}

local p = producer:new(broker_list, { producer_type = "async" })
local ok, err = p:send("app-log-topic", nil, original_body)
```

`lua-resty-kafka`를 쓰려면 의존 모듈이 필요하다.

```bash
luarocks install lua-cjson
luarocks install penlight
```

개발 중에 두 가지 이슈를 만났다.

1. **Kafka 메시지에 timestamp가 안 남는 문제**: Kafka API version을 1에서 2로 올리니 해결됐다
2. **Kafka metadata 조회 실패**: API version 2에서 발생해서 다시 1로 내렸다

결국 메시지 발행은 API version 2, metadata 조회는 API version 1을 쓰는 식으로 분리 처리했다.

---

## 아키텍처 변천사

처음 설계했던 구조와 최종 구조가 상당히 다르다.

### 초기 설계: 프록시 → Kafka → Flink → 해외 서버

```
앱 → APISIX → Kafka → Flink(비식별화) → 해외 트래킹 서버
```

프록시는 Kafka로만 보내고 비식별화는 Flink에서 처리하는 구조였다. 문제는 해외 서버가 HTTP 400 에러를 내릴 때 그걸 클라이언트에 전달할 방법이 없다는 것이다. Kafka로 보내는 순간 응답은 끊기니까.

### 최종 설계: 프록시에서 직접 비식별화

```
앱 → APISIX → (원본 → Kafka)
             → (해싱 후 → 해외 트래킹 서버 → 응답 → 앱)
```

프록시에서 직접 비식별화를 수행하고 해외 서버로 전송한다. 장단점은 명확하다.

**장점:**
- 해외 서버의 응답(400 포함)을 클라이언트에 그대로 전달 가능
- Flink 애플리케이션이 필요 없어짐
- 실시간 처리

**단점:**
- 프록시 서버에 부하가 집중됨
- request body 파싱 + 해싱 + Kafka 발행을 한 요청 안에서 처리

부하 집중이 걱정됐지만 APISIX의 성능이 충분히 받쳐줬다. 자세한 수치는 아래에서 다룬다.

---

## 부하 테스트

### 테스트 환경

nGrinder로 부하를 걸었다. 프록시 서버 스펙은 Pod당 2 core, 4GiB 메모리.

### 코어당 처리량

APISIX 공식 문서에는 1 core당 10,000 QPS를 처리할 수 있다고 나와 있다. 하지만 우리처럼 request body를 파싱하고 해싱하는 Lua 스크립트가 붙으면 그보다 낮아진다.

실측 결과 (2 core 기준, 1분 30초씩 테스트):

| Vuser | Peak TPS | CPU 사용률 | 에러 |
|-------|----------|-----------|------|
| 990 | 2,381 | 32% | 0 |
| 1,980 | 4,907 | 48% | 1 |
| 3,960 | 7,000 | 99% | 0 |

안정적인 운영을 위해서는 2 core 기준 **5,000 TPS** 정도가 적당하다. 우리의 피크 트래픽이 약 55,000 TPS이므로 Pod 12개면 커버된다.

### 튜닝 포인트

**Keepalive 설정**: 적용 전에는 p95 레이턴시가 요청마다 흔들렸는데 keepalive 설정 후 **p95 기준 10ms 이하**로 안정화됐다. warm-up 이후에는 1ms 미만도 나왔다.

**nginx worker_connections**: APISIX의 기본값은 `auto`인데 이게 Pod 코어 수가 아니라 **노드 코어 수**를 기준으로 worker를 할당한다. 과도한 worker가 CPU 병목을 일으킬 수 있으므로 Pod 코어 수에 맞춰 수동 설정했다. 효과는 근소하지만 CPU 사용률이 내려가면서 TPS는 올라갔다.

**스케일 아웃 전략**: 급격한 트래픽이 들어오면 스케일 아웃되기 전에 CPU가 100%를 찍으면서 에러가 터진다. 스트리밍과 달리 백프레셔 메커니즘이 없기 때문이다. 두 가지로 대응했다.

1. **KEDA로 CPU 50% 기준 스케일 아웃** — 여유 있게 잡아야 한다
2. **스케줄 기반 스케일링** — 피크 타임(점심, 저녁)에 맞춰 미리 Pod를 늘려 놓는다

### 502 에러 원인

테스트 중 간헐적으로 502가 발생했다. upstream 서버에서 response header를 읽다가 타임아웃이 걸린 것이다. 원인은 복합적이었다.

- 해외 서버 스테이징 환경이 spot 인스턴스로 구성되어 있어 응답이 불안정
- 급격한 트래픽에서 스케일 아웃 전 CPU가 포화
- nGrinder agent당 vuser를 과하게 잡으면 클라이언트 쪽 네트워크 병목이 생겨 서버 응답이 느려 보이는 현상

운영 환경에서는 min Pod를 보수적으로 잡고 스케줄 기반 스케일링을 병행하니 502가 사라졌다.

---

## Kubernetes 배포

### Helm 차트 구성

APISIX를 **decoupled 모드**로 배포했다. control-plane과 data-plane을 분리하는 방식이다.

- **control-plane** (apisix-control): etcd와 함께 배포. 라우팅 설정을 관리한다
- **data-plane** (apisix-data): 실제 트래픽을 처리한다. `externalEtcd` 설정으로 control-plane의 etcd에 연결한다

etcd는 PVC(gp3)로 데이터를 영구 보관한다. 표준 EKS의 기본 스토리지 클래스가 gp3이므로 별도 설정 없이 PVC를 생성하면 된다.

### HPA 설정 (KEDA)

```yaml
# KEDA ScaledObject (요약)
triggers:
  - type: prometheus
    metadata:
      query: avg(rate(container_cpu_usage_seconds_total{...}[1m])) * 100
      threshold: "50"
  - type: memory
    metadata:
      value: "80"
```

CPU 50% 또는 메모리 80%를 넘으면 스케일 아웃한다. 여기에 스케줄 기반 스케일링을 더해서 피크 타임에 미리 Pod를 확보한다.

### 보안

퍼블릭으로 노출되는 리버스 프록시 ALB에 **AWS WAF**를 연동했다. 정보보안팀 검토 결과 WAF만 적용하면 클라이언트 보안에 문제없다는 결론이 나왔다.

---

## 모니터링

Grafana 대시보드를 구성해서 다음 지표를 실시간으로 본다.

- **System**: CPU 사용률, 메모리 사용률 (Pod별)
- **Nginx**: 총 요청 수, accepted/handled connections, connection state
- **HTTP**: 상태 코드별 RPS, 서비스/라우트별 RPS
- **Latency**: APISIX 레이턴시, upstream 레이턴시, 전체 요청 레이턴시
- **Bandwidth**: 서비스/라우트별 ingress/egress
- **etcd**: modify indexes, reachable 상태

APISIX의 라우트 설정에 `prometheus` 플러그인을 추가해야 request 단위 메트릭이 수집된다. 이걸 빠뜨리면 시스템 메트릭만 나오고 HTTP 관련 지표가 비어 있다.

알럿은 Grafana → OpsGenie → Slack 채널로 연동했다.

---

## 운영 결과

약 3개월간 운영한 결과를 정리하면 이렇다.

| 지표 | 목표 | 실제 |
|------|------|------|
| 비용 절감 | 기존 대비 20% | **29.8% 절감** |
| 레이턴시 (p95) | 40ms | **25ms** |
| 가용성 | 99.99% | **100%** |
| PII 비식별화 성공률 | 100% | **100%** |

비용 절감이 목표보다 높았던 건 Flink 애플리케이션이 빠지면서다. 프록시에서 직접 비식별화를 처리하니 별도 스트림 프로세싱 비용이 사라졌다.

운영 초기에 499 에러(클라이언트가 연결을 끊는 경우)와 408 에러(서버 타임아웃)가 소량 발생했다. 400 에러를 제외한 나머지는 SDK에서 재전송 처리하므로 데이터 유실은 없었다.

---

## 마치며

APISIX를 프록시 서버로 쓰면서 배운 것을 정리한다.

1. **API Gateway를 프록시로 쓰는 게 자연스러울 때가 있다.** request body 조작과 HTTP 응답 전달이 필요하면 로그 수집기보다 API Gateway가 맞다.
2. **커스텀 플러그인은 피할 수 없다.** 기존 플러그인 조합으로 해결하려고 붙잡지 말고 빨리 커스텀 플러그인을 만드는 게 낫다. APISIX의 플러그인 구조가 잘 되어 있어서 개발 자체는 어렵지 않다.
3. **스케일 아웃보다 미리 확보하는 게 낫다.** 급격한 트래픽에서 리액티브 스케일링은 늦다. 스케줄 기반으로 피크 타임에 미리 Pod를 올려놓는 게 안정적이다.
4. **nginx worker 수를 확인하라.** APISIX의 `auto` 설정이 Pod가 아닌 노드 기준으로 worker를 만든다. 컨테이너 환경에서는 수동으로 맞춰야 한다.

**참고 자료:**
- [Apache APISIX 공식 문서](https://apisix.apache.org/docs/)
- [lua-resty-kafka (GitHub)](https://github.com/doujiang24/lua-resty-kafka)
- [APISIX Custom Plugin Development](https://apisix.apache.org/docs/apisix/plugin-develop/)
- [Apache APISIX Grafana Dashboard](https://grafana.com/grafana/dashboards/11719-apache-apisix/)
