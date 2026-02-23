---
title: "Building a Proxy Server with Apache APISIX: De-identifying 1.3 Billion Daily Logs"
date: 2026-02-24
draft: false
categories: [Data Engineering]
tags: [apisix, proxy, lua, kubernetes, privacy, kafka]
showTableOfContents: true
summary: "We needed to strip PII from app logs before sending them to an overseas tracking server. This post covers our journey from Fluent-bit to Kong to APISIX, custom Lua plugin development, 55K TPS load testing, and Kubernetes deployment."
---

We had to send app logs to an overseas tracking server. Problem: the logs contained personally identifiable information — user IDs, device IDs, order numbers. Privacy law says you can't send that abroad in plaintext.

The fix: put a proxy in the middle. It receives logs from the app, hashes PII fields, forwards the sanitized data overseas. At the same time it publishes the original data to an internal Kafka cluster so teams like ads and recommendations can still use it.

This post documents how we built that proxy with Apache APISIX, handling 1.3 billion logs per day.

---

## Why APISIX

### We Started with Fluent-bit

Log collection, so Fluent-bit felt natural. But once we listed the actual requirements, it was clear we needed an **API gateway**, not a log forwarder.

- Parse and modify HTTP request bodies (hash PII fields)
- Pass the upstream server's responses (including 400 errors) back to the client
- Publish original data to Kafka at the same time

Fluent-bit can't do any of that cleanly.

### Kong vs APISIX

Both are nginx-based and support Lua scripting. We tried Kong first. It failed at the Helm chart deployment stage due to a [known issue](https://github.com/Kong/charts/issues/1198).

APISIX won for three reasons:

- **Lightweight**: Lower memory footprint and higher throughput per core than Kong
- **Declarative routing**: Send config via Admin API, it gets stored in etcd permanently
- **Clean plugin pipeline**: Requests flow through rewrite → access → header_filter → body_filter → log stages. Easy to inject logic at exactly the right point

---

## Custom Plugin Development

### Why Built-in Plugins Weren't Enough

APISIX ships with a `kafka-logger` plugin that sends request data to Kafka. The catch: if we run our de-identification plugin (`encrypt-pii`) first, kafka-logger sees the **hashed data, not the original**.

What we actually needed:

```
App → APISIX → (1) Publish original to Kafka
              → (2) Hash PII fields
              → (3) Forward hashed data to overseas server
              → (4) Return server's response to app
```

Order matters. Original goes to Kafka first, then we hash. No existing plugin combo guarantees that ordering, so we built a **single custom plugin** combining kafka-logger and de-identification.

### PII Hashing in Lua

APISIX runs on LuaJIT. We used `lua-resty-string` for SHA-256:

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

Three fields get hashed: user ID, device ID, and order number. The plugin parses the JSON body, finds target fields, hashes them, and re-serializes.

### Kafka Publishing

Same plugin uses `lua-resty-kafka` to send the original body to Kafka:

```lua
local producer = require "resty.kafka.producer"

local broker_list = {
    { host = "kafka-broker-01", port = 9092 },
}

local p = producer:new(broker_list, { producer_type = "async" })
local ok, err = p:send("app-log-topic", nil, original_body)
```

Dependencies needed:

```bash
luarocks install lua-cjson
luarocks install penlight
```

We hit two issues during development:

1. **No timestamp in Kafka messages**: Fixed by bumping Kafka API version from 1 to 2
2. **Metadata fetch failures**: Happened on API version 2, so we dropped back to 1

Ended up using API version 2 for message publishing and version 1 for metadata queries.

---

## Architecture Evolution

The final design looked nothing like our first sketch.

### Initial Design: Proxy → Kafka → Flink → Overseas

```
App → APISIX → Kafka → Flink (de-identification) → Tracking Server
```

Proxy just forwards to Kafka. Flink handles PII hashing and sends data onward. Clean separation, but one fatal flaw: when the tracking server returns HTTP 400 for a bad request, there's no way to pass that back to the client. Once you hand off to Kafka, the response path is gone.

### Final Design: Proxy Does Everything

```
App → APISIX → (original → Kafka)
             → (hashed → Tracking Server → response → App)
```

The proxy handles de-identification directly and forwards to the overseas server.

**Upside:**
- Client gets the tracking server's actual response, 400s included
- No Flink application needed
- Real-time processing

**Downside:**
- All load concentrates on the proxy
- Body parsing + hashing + Kafka publish happens in a single request

The load concern was real, but APISIX handled it. Numbers below.

---

## Load Testing

### Setup

We used nGrinder. Proxy pods ran on 2 cores, 4GiB memory each.

### Throughput Per Core

APISIX docs claim 10,000 QPS per core. With our Lua scripting overhead (body parsing, hashing, Kafka publish), real numbers are lower.

Measured results (2 cores per pod, 90-second runs):

| Vusers | Peak TPS | CPU Usage | Errors |
|--------|----------|-----------|--------|
| 990 | 2,381 | 32% | 0 |
| 1,980 | 4,907 | 48% | 1 |
| 3,960 | 7,000 | 99% | 0 |

**Safe operating point: ~5,000 TPS on 2 cores.** Our peak traffic hits 55,000 TPS, so 12 pods cover it.

### Tuning That Mattered

**Keepalive**: Before enabling it, p95 latency jumped around on every request. After: **p95 under 10ms**, often under 1ms after warm-up.

**nginx worker_connections**: APISIX defaults to `auto`, which counts **node cores, not pod cores**. In containers this creates way too many workers and causes CPU contention. Set it manually to match your pod's core count. Small gain, but measurable — CPU dropped while TPS went up.

**Scale-out strategy**: When traffic spikes hit, reactive scaling is too slow. CPU pegs at 100% before new pods come up, and errors start flying. Unlike stream processing, there's no backpressure here. We used two approaches:

1. **KEDA with CPU threshold at 50%** — aggressive, on purpose
2. **Schedule-based scaling** — pre-scale before known peak times (lunch, dinner)

### 502 Errors

Intermittent 502s showed up during testing. Root causes:

- The overseas server's staging environment ran on spot instances — responses were flaky
- CPU saturation before scale-out kicked in
- nGrinder artifact: too many vusers per agent causes client-side network bottlenecks that make server latency look worse than it is

In production with conservative min-pod counts and schedule-based scaling, 502s disappeared.

---

## Kubernetes Deployment

### Helm Chart

We deployed APISIX in **decoupled mode** — separate control-plane and data-plane.

- **Control-plane** (apisix-control): Deploys with etcd. Manages routing config
- **Data-plane** (apisix-data): Handles actual traffic. Connects to control-plane's etcd via `externalEtcd` config

etcd uses a PVC (gp3) for persistent storage. Standard EKS defaults to gp3, so no extra config needed.

### Autoscaling (KEDA)

```yaml
# KEDA ScaledObject (simplified)
triggers:
  - type: prometheus
    metadata:
      query: avg(rate(container_cpu_usage_seconds_total{...}[1m])) * 100
      threshold: "50"
  - type: memory
    metadata:
      value: "80"
```

Scale out at 50% CPU or 80% memory. Combined with schedule-based scaling for peak hours.

### Security

The public-facing reverse proxy ALB got **AWS WAF** attached. Our security team reviewed it and confirmed WAF alone was sufficient for client-facing protection.

---

## Monitoring

We built a Grafana dashboard tracking:

- **System**: CPU and memory utilization per pod
- **Nginx**: Total requests, accepted/handled connections, connection state
- **HTTP**: RPS by status code, RPS per service/route
- **Latency**: APISIX latency, upstream latency, total request latency
- **Bandwidth**: Ingress/egress per service and route
- **etcd**: Modify indexes, reachability

One gotcha: you need to add the `prometheus` plugin to your route config for per-request metrics. Without it you only get system-level metrics and all the HTTP dashboards stay empty.

Alerts flow through Grafana → OpsGenie → Slack.

---

## Production Results

After three months of operation:

| Metric | Target | Actual |
|--------|--------|--------|
| Cost reduction | 20% vs. previous | **29.8% reduction** |
| Latency (p95) | 40ms | **25ms** |
| Availability | 99.99% | **100%** |
| PII de-identification | 100% | **100%** |

Cost beat the target because cutting Flink out of the pipeline eliminated stream processing costs entirely.

Early on we saw some 499 errors (client dropped connection) and 408 errors (server timeout). The SDK retries everything except 400s, so no data was lost.

---

## Takeaways

1. **Sometimes an API gateway is the right proxy.** If you need to modify request bodies and relay HTTP responses, a log forwarder won't cut it. Reach for an API gateway.
2. **Build the custom plugin early.** Don't waste time trying to chain existing plugins into doing something they weren't designed for. APISIX's plugin structure makes custom development straightforward.
3. **Pre-scale beats reactive scaling.** For bursty HTTP traffic with no backpressure, schedule-based scaling before peak hours is more reliable than waiting for KEDA to react.
4. **Check your nginx worker count.** APISIX's `auto` setting looks at node cores, not pod cores. In containers, set it manually or you'll get CPU contention from too many workers.

**References:**
- [Apache APISIX Documentation](https://apisix.apache.org/docs/)
- [lua-resty-kafka (GitHub)](https://github.com/doujiang24/lua-resty-kafka)
- [APISIX Custom Plugin Development](https://apisix.apache.org/docs/apisix/plugin-develop/)
- [Apache APISIX Grafana Dashboard](https://grafana.com/grafana/dashboards/11719-apache-apisix/)
