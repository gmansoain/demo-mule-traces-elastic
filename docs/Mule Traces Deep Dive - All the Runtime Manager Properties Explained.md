---
title: "Mule Traces Deep Dive: All the Runtime Manager Properties Explained"
slug: mule-traces-deep-dive-runtime-manager-properties
description: Master every OpenTelemetry property for Direct Telemetry Streams in Mule. Learn how to configure tracing, sampling, batching, and service identification for production observability.
author: Gonzalo Marcos
date: 2026-04-27
lang: en
category: observability
tags:
  - runtime-fabric
  - observability
  - splunk
  - opentelemetry
  - logging
  - deep-dive
  - english
  - cloudhub-2
type: deep-dive
difficulty: intermediate
read_time: 18
mule_version: "4.11"
platform:
  - runtime-fabric
status: draft
series: Direct Telemetry Streams for Mule
series_part: 2
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-FF9900) ![Mule 4.x|97](https://img.shields.io/badge/Mule_Runtime-4.11+-00A0DF?logo=mulesoft&logoColor=white) ![Runtime Fabric](https://img.shields.io/badge/Platform-Runtime_Fabric-00A0DF?logo=mulesoft&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Deep Dive](https://img.shields.io/badge/Type-Deep_Dive-8e44ad) ![Series](https://img.shields.io/badge/Series-Direct_Telemetry_Streams-16a085) ![Part 2](https://img.shields.io/badge/Part-2-16a085) ![18 min](https://img.shields.io/badge/Read_Time-18_min-lightgrey)

> 🔗 **Previous:** [Part 1 — Export Mule RTF App Logs to Splunk with the OTel Collector](Export%20Mule%20RTF%20App%20Logs%20to%20Splunk%20with%20the%20OTel%20Collector.md) · **Series:** Direct Telemetry Streams for Mule

#blog 
# Mule Traces Deep Dive: All the Runtime Manager Properties Explained

In the previous post, [Export Mule RTF App Logs to Splunk with the OTel Collector](Export%20Mule%20RTF%20App%20Logs%20to%20Splunk%20with%20the%20OTel%20Collector.md) we connected a Mule test app deployed on Runtime Fabric to Splunk Enterprise using just four Runtime Manager properties. That was enough to get traces flowing. Now we'll go further.

The Direct Telemetry Stream exposes a full set of properties that control how traces are generated, sampled, batched, compressed, retried, and identified. Each one gives us a lever. In this post, we'll cover every property, explain what it does, why it matters, and give concrete examples we can test in our app right now.

We'll group the properties into four areas: **Exporter Setup**, **Tracing Level**, **Sampling**, and **Export Tuning** (which covers batching, backpressure, retry, timeout, and compression). We'll finish with the **Service Identification** properties.

---

## Where We Set These Properties

All properties go in **Runtime Manager → Applications → Settings → Properties**. For Runtime Fabric deployments, we set them as application-level properties. They take effect on the next deployment or redeploy.

We never hardcode these in our Mule app's `mule-artifact.json` or `log4j2.xml`. They belong in Runtime Manager so we can change them without a code change or a new artifact build.

---

## Group 1 — Exporter Setup

These four properties define the basic connection between the Mule runtime and the OTel Collector. Without them configured correctly, no data flows.

---

### `mule.openTelemetry.tracer.exporter.enabled`

|Default|`false`|
|---|---|
|Required|Yes, to activate the feature|

This is the master switch. The exporter is off by default. We set it to `true` to start exporting traces.

**Why it matters.** The default is `false` on purpose. Enabling the exporter adds processing overhead to every flow execution. We choose when to turn it on.

**Recommendation.** Enable it in all non-production environments permanently. In production, enable it but pair it with a low sampling rate (covered in Group 3). Never leave it `true` in production with a 100% sampling rate.

**Example:**

```
mule.openTelemetry.tracer.exporter.enabled=true
```

---

### `mule.openTelemetry.tracer.exporter.type`

|Default|`GRPC`|
|---|---|
|Available values|`GRPC`, `HTTP`|

This sets the transport protocol used to send spans to the OTel Collector endpoint. `GRPC` uses port `4317`. `HTTP` uses port `4318`.

**Why it matters.** Our OTel Collector must listen on the matching protocol and port. A mismatch means silent delivery failure — the runtime will log export errors, but no spans will arrive at the collector.

**Recommendation.** Use `HTTP` for simplicity in test environments. GRPC is more efficient at high throughput due to binary framing and multiplexing, so it is worth evaluating in production when trace volume is high. In our test setup from the previous post, we deployed the collector with the OTLP HTTP receiver on port `4318`, so we use `HTTP` here.

**Example:**

```
mule.openTelemetry.tracer.exporter.type=HTTP
```

---

### `mule.openTelemetry.tracer.exporter.endpoint`

|Default|`http://localhost:4317` (GRPC) or `http://localhost:4318/v1/traces` (HTTP)|
|---|---|
|Required|Yes, to point to our actual collector|

This is the URL where the Mule runtime sends traces. The default points to `localhost`, which is never correct in a containerized or K8s environment.

**Why it matters.** This is the most common misconfiguration. If we forget to set this, spans go to `localhost` inside the Mule pod and are silently dropped.

**Recommendation.** Always set this explicitly. Use the internal K8s service DNS name when the OTel Collector runs in the same cluster as RTF. This avoids any external network dependency.

**Example — internal K8s service (our setup from the previous post):**

```
mule.openTelemetry.tracer.exporter.endpoint=http://otel-collector-opentelemetry-collector.otel-collector.svc.cluster.local:4318/v1/traces
```

---

### `mule.openTelemetry.tracer.exporter.headers`

|Default|Empty|
|---|---|
|Format|Comma-separated `key=value` pairs|

This injects custom HTTP headers into every export call the runtime makes to the OTel Collector endpoint.

**Why it matters.** Some collector or backend configurations require an authentication header or a routing header on every incoming request. Without this property, we can't pass those headers from the Mule runtime.

**Recommendation.** Use this when our OTel Collector or backend requires token-based authentication on the OTLP endpoint itself — for example, when we expose the collector outside the cluster and need to protect it. For our internal K8s setup, this property is not needed. Authentication lives at the collector-to-Splunk HEC layer, not at the Mule-to-collector layer.

**Example — passing an auth token to a protected collector endpoint:**

```
mule.openTelemetry.tracer.exporter.headers=Authorization=Bearer my-token,X-Tenant-ID=acme
```

---

## Group 2 — Tracing Level

This single property has a large impact on how much data the runtime generates. It controls the **depth** of instrumentation — how many spans Mule creates for a given flow execution.

---

### `mule.openTelemetry.tracer.level`

| Default          | `MONITORING`                      |
| ---------------- | --------------------------------- |
| Available values | `OVERVIEW`, `MONITORING`, `DEBUG` |

Mule assigns every span a tracing level. This property sets the threshold. Spans at or below the configured level are generated and exported. Spans above the threshold are silently skipped.

Here is what each level captures:

|Level|What Gets a Span|
|---|---|
|`OVERVIEW`|Flow spans only. One span per flow execution.|
|`MONITORING`|Flow spans + individual component spans (Set Variable, Transform Message, HTTP Requester, etc.). This is the default.|
|`DEBUG`|Everything in `MONITORING` plus internal runtime spans: parameter resolution, connection acquisition, and other low-level operations.|

**Why it matters.** Each additional span consumes memory in the export queue, network bandwidth to the collector, and storage in Splunk. A long flow with ten components at `DEBUG` level can generate dozens of spans per request. At scale, this adds up fast.

**Recommendation.** Leave it at `MONITORING` in all normal operations. This gives us full visibility into flow execution without the noise of internal runtime mechanics. Switch to `DEBUG` temporarily when troubleshooting a specific performance or execution problem. Never use `DEBUG` in production permanently.

**Example — enable DEBUG temporarily for troubleshooting:**

```
mule.openTelemetry.tracer.level=DEBUG
```

**What to look for in Splunk at each level.**

At `OVERVIEW`, each search result in Splunk represents one full flow execution. We'll see one event per request.

At `MONITORING`, we'll see one event per component in the flow. For a flow with five components, we'll see five events sharing the same `traceId`.

At `DEBUG`, we'll see additional events for internal operations. Expect ten or more events per component in complex processors like the HTTP Requester or Database connector.

**Test this in our app.** We'll set the property to `OVERVIEW`, trigger a request, and count the events in Splunk:

```
index="main" sourcetype="mule:otel:trace" traceId="<our-trace-id>"
```

We'll then change it to `MONITORING`, redeploy, trigger another request, and compare the event count for the same flow.

---

## Group 3 — Sampling

Sampling controls **how many** traces the Mule runtime generates and exports. It is our most powerful cost-control lever.

---

### `mule.openTelemetry.tracer.exporter.sampler`

|Default|`parentbased_traceidratio`|
|---|---|
|Available values|`always_on`, `always_off`, `traceidratio`, `parentbased_always_on`, `parentbased_always_off`, `parentbased_traceidratio`|

This sets the sampling algorithm. The algorithm decides, at the moment a message source fires, whether that execution will be traced at all.

Mule uses **head sampling** — the decision is made once, at the start of the trace, and applies to every span in that execution. This is efficient and simple but has a trade-off: we can't retroactively sample based on whether an error occurred later in the flow.

Here is what each algorithm does:

**`always_on`** — Samples every single trace. Use this only for testing or very low-traffic apps. In production this is expensive.

**`always_off`** — Samples nothing. All tracing is disabled regardless of the `enabled` flag. This is useful to quickly suspend tracing without redeploying or disabling the exporter entirely.

**`traceidratio`** — Samples a percentage of traces based on the trace ID. The percentage is set by `sampler.arg`. The decision is deterministic — the same trace ID always gets the same decision. All spans within a sampled trace are consistently included.

**`parentbased_traceidratio`** — This is the default and the recommended production sampler. It works like `traceidratio` for root spans (spans with no parent). When a parent span exists — for example, when our Mule app receives a request from another service that already has a trace context header — it inherits the parent's sampling decision. This preserves trace continuity across distributed systems.

**`parentbased_always_on`** — Always samples root spans. Inherits the parent decision when a parent exists.

**`parentbased_always_off`** — Never samples root spans. Still participates in traces started by others. Use this for services that should only appear as part of traces initiated elsewhere.

**Recommendation.** Keep the default `parentbased_traceidratio` in production. It respects W3C Trace Context headers from upstream callers. Pair it with a sensible `sampler.arg` value.

**Example — switch to always_off to pause tracing without disabling the exporter:**

```
mule.openTelemetry.tracer.exporter.sampler=always_off
```

Tail sampling supports more advanced cases but is costly and can affect performance because the algorith takes the smpling decision by considering the trace content. 

---

### `mule.openTelemetry.tracer.exporter.sampler.arg`

|Default|`0.1`|
|---|---|
|Range|`0` to `1`|

This sets the sampling percentage for ratio-based samplers. `0.1` means 10%. `1` means 100%. `0` means 0% (equivalent to `always_off`).

**Why it matters.** This is the most direct dial for controlling trace volume and cost. A value of `0.1` on a service handling 1,000 requests per minute exports traces for 100 of them. A value of `1` exports all 1,000.

**Recommendation.** Use `1` in test environments only. In production, start at `0.1` (10%) and adjust based on actual volume, Splunk storage cost, and observability requirements. For low-traffic services (fewer than 100 requests per minute), `1` is often acceptable in production.

**Example — 10% sampling for production:**

```
mule.openTelemetry.tracer.exporter.sampler.arg=0.1
```

**Example — 100% sampling for test:**

```
mule.openTelemetry.tracer.exporter.sampler.arg=1
```

**Test this in our app.** We'll set `sampler.arg=0.5` and send 10 requests. We'll count how many traces appear in Splunk. We should see roughly 5 — the exact number varies because the decision is probabilistic per trace ID, not a strict alternating pattern.

---

## Group 4 — Export Tuning

These properties control the mechanics of how spans leave the Mule runtime: how they are grouped, how long the runtime waits before sending, what happens when things go wrong, and how the runtime handles pressure.

---

### `mule.openTelemetry.tracer.exporter.timeout`

| Default | `10` seconds |

This is the maximum time the Mule runtime waits for the OTel Collector to acknowledge a single export call. If the collector does not respond within this window, the export call times out.

**Why it matters.** If our OTel Collector is under pressure or temporarily unavailable, slow export responses can hold up the export thread. A short timeout prevents this. A very short timeout risks false failures if the collector is legitimately slow under load.

**Recommendation.** The default of 10 seconds is generous. In a stable internal K8s environment where the collector is co-located in the same cluster, we can lower this to `5s` to fail faster. We should never set it below `2s`.

**Example:**

```
mule.openTelemetry.tracer.exporter.timeout=5s
```

---

### `mule.openTelemetry.tracer.exporter.compression`

|Default|`none`|
|---|---|
|Available values|`none`, `gzip`|

This compresses the trace payload before sending it to the OTel Collector.

**Why it matters.** Span data is repetitive JSON-like content. Gzip compression typically reduces payload size by 70–80%. This matters when the Mule runtime sends spans over a network with bandwidth constraints, or when trace volume is high.

**Recommendation.** Use `none` in test environments — the overhead is not worth it for small volumes. Enable `gzip` in production when trace volume is high or the network path to the collector has limited bandwidth. Compression adds a small CPU cost on the Mule runtime, so always test the performance impact before enabling it in production.

**Example:**

```
mule.openTelemetry.tracer.exporter.compression=gzip
```

---

### Batch Properties

The batch properties control how the Mule runtime groups spans before sending them. Batching reduces the number of calls to the OTel Collector and improves overall throughput.

---

#### `mule.openTelemetry.tracer.exporter.batch.scheduledDelay`

| Default | `5000` milliseconds |

This is the maximum time the runtime waits before flushing a batch, even if the batch is not full. Think of it as a flush timer. Every 5 seconds (by default), whatever spans have accumulated in the queue are sent.

**Why it matters.** This property directly controls trace latency in Splunk. If we set it to `5000ms`, we wait up to 5 seconds after a flow executes before its spans appear in Splunk. For real-time troubleshooting, that delay is noticeable.

**Recommendation.** Lower this to `1000ms` or `2000ms` in test environments where we want near-real-time visibility. In production, the default `5000ms` is a good balance between latency and efficiency. We should not set it below `500ms` — very frequent flushes increase HTTP call overhead on both the runtime and the collector.

**Example — faster flush for test:**

```
mule.openTelemetry.tracer.exporter.batch.scheduledDelay=1000
```

---

#### `mule.openTelemetry.tracer.exporter.batch.maxSize`

| Default | `512` spans |

This is the maximum number of spans in a single export batch. When the batch reaches this size, it flushes immediately — before the `scheduledDelay` timer fires.

**Why it matters.** Under high traffic, the export queue fills up fast. A large `maxSize` means fewer, larger HTTP calls. A small `maxSize` means more frequent, smaller calls.

**Recommendation.** The default of `512` works well for most workloads. Increase it only if we observe a very high rate of export calls and want to reduce HTTP overhead. Decrease it only if the collector struggles with large payloads.

**Example:**

```
mule.openTelemetry.tracer.exporter.batch.maxSize=512
```

---

#### `mule.openTelemetry.tracer.exporter.batch.queueSize`

| Default | `2048` spans |

This is the total number of spans the runtime holds in memory while waiting to be batched and exported. It acts as a buffer between the flow execution and the export pipeline.

**Why it matters.** This is our safety net. When the OTel Collector is slow or temporarily unavailable, spans accumulate in this queue instead of being dropped immediately. A larger queue gives the collector more time to recover. A smaller queue uses less memory.

**Recommendation.** The default of `2048` is sufficient for most test apps. In production, size this queue based on the product of our peak spans-per-second rate and the maximum acceptable collector recovery time. For example: if our app generates 100 spans per second and we want to buffer up to 30 seconds of recovery time, we need a queue size of at least 3,000. Keep an eye on the warning logs — `Export queue overflow: N spans have been dropped` is the signal to increase this value.

**Example — larger buffer for high-traffic production service:**

```
mule.openTelemetry.tracer.exporter.batch.queueSize=4096
```

---

#### `mule.openTelemetry.tracer.exporter.batch.backPressure.strategy`

|Default|`DROP`|
|---|---|
|Available values|`DROP`, `BLOCK`|

This controls what happens when the export queue defined by `queueSize` is full and a new span arrives.

**`DROP`** — The new span is discarded. The flow execution continues without interruption. A warning log records the drop: `Export queue overflow: N spans have been dropped.`

**`BLOCK`** — The flow execution thread waits until space opens in the queue before continuing. No spans are lost, but flow processing stalls.

**Why it matters.** This is a fundamental trade-off between **data completeness** and **application availability**. `DROP` protects the Mule app from collector slowness at the cost of losing trace data. `BLOCK` preserves all trace data at the cost of slowing down the app.

**Recommendation.** Use `DROP` in all cases. Our Mule app serves business traffic. Trace data is observability data — it is important, but it must never slow down or block the processing of real requests. If we find ourselves dropping spans regularly, the right fix is to increase `queueSize`, tune the collector, or lower the sampling rate — not to switch to `BLOCK`.

**Example:**

```
mule.openTelemetry.tracer.exporter.batch.backPressure.strategy=DROP
```

---

### Retry / Backoff Properties

When an export call fails with a retryable error — for example, the OTel Collector returns an HTTP 503 — the Mule runtime retries the export using an exponential backoff with jitter. These four properties tune that retry behavior.

---

#### `mule.openTelemetry.tracer.exporter.backoff.maxAttempts`

| Default | `5` |

The total number of export attempts before the runtime gives up on a batch of spans.

**Recommendation.** The default of `5` is suitable for most environments. Increasing this gives the collector more time to recover but holds spans in memory longer. Decreasing it fails faster and frees memory sooner.

**Example:**

```
mule.openTelemetry.tracer.exporter.backoff.maxAttempts=5
```

---

#### `mule.openTelemetry.tracer.exporter.backoff.initial`

| Default | `1` second |

The wait time before the first retry attempt after a failed export.

**Recommendation.** The default of `1s` is appropriate. Setting this too low floods the collector with retries before it has time to recover.

**Example:**

```
mule.openTelemetry.tracer.exporter.backoff.initial=1s
```

---

#### `mule.openTelemetry.tracer.exporter.backoff.maximum`

| Default | `5` seconds |

The maximum wait time between any two retry attempts, regardless of what the multiplier calculates.

**Recommendation.** The default of `5s` caps the backoff at a reasonable ceiling. We can increase this if the collector takes longer to recover from failures in our environment.

**Example:**

```
mule.openTelemetry.tracer.exporter.backoff.maximum=5s
```

---

#### `mule.openTelemetry.tracer.exporter.backoff.multiplier`

| Default | `1.5` |

The factor applied to the previous wait time to calculate the next retry delay. With a `1s` initial delay and a `1.5` multiplier, retries happen at 1s, 1.5s, 2.25s, 3.37s, and 5s (capped by `maximum`).

**Recommendation.** The default of `1.5` produces a sensible exponential curve. We rarely need to change this.

**Example:**

```
mule.openTelemetry.tracer.exporter.backoff.multiplier=1.5
```

---

## Group 5 — Service Identification

These two properties tag every span and log with a service name and namespace. They are how we tell Splunk which app a span came from.

---

### `mule.openTelemetry.exporter.resource.service.name`

| Default | Artifact ID of the Mule app |

This sets the `service.name` attribute on every span the app exports. By default it equals the artifact ID we define in `pom.xml`. When we search in Splunk, this field tells us which app a span belongs to.

**Why it matters.** In a multi-app environment where many Mule apps send traces to the same OTel Collector and Splunk index, this is how we filter by application. Without a clear name, search results from multiple apps mix together.

**Recommendation.** The default (artifact ID) is usually good enough. Override it when the artifact ID is cryptic or does not match the service name used by the wider ops team.

**Example — override to a human-readable name:**

```
mule.openTelemetry.exporter.resource.service.name=hello-world-api
```

**Splunk search using service name:**

```
index="main" sourcetype="mule:otel:trace" resource.service.name="hello-world-api"
```

---

### `mule.openTelemetry.exporter.resource.service.namespace`

| Default | `mule` |

This sets the `service.namespace` attribute on every span. It groups services into a logical namespace — useful for separating business domains, teams, or environments in Splunk searches.

**Why it matters.** When multiple teams share the same Splunk index, the namespace lets each team filter to their own services without knowing every service name. It also helps when the same app name exists in different environments.

**Recommendation.** Override this to reflect our naming conventions — for example, the team name, business domain, or environment. Use consistent values across all apps in the same domain.

**Example — tag spans with a domain and environment:**

```
mule.openTelemetry.exporter.resource.service.namespace=payments-dev
```

**Splunk search filtering by namespace:**

```
index="main" sourcetype="mule:otel:trace" resource.service.namespace="payments-dev"
```

---

## A Complete Reference Configuration

Below is a full set of properties with recommended values for two scenarios: a test environment and a production starting point.

**Test environment — maximum visibility, no cost concern:**

```
mule.openTelemetry.tracer.exporter.enabled=true
mule.openTelemetry.tracer.exporter.type=HTTP
mule.openTelemetry.tracer.exporter.endpoint=http://otel-collector-opentelemetry-collector.otel-collector.svc.cluster.local:4318/v1/traces
mule.openTelemetry.tracer.level=MONITORING
mule.openTelemetry.tracer.exporter.sampler=parentbased_traceidratio
mule.openTelemetry.tracer.exporter.sampler.arg=1
mule.openTelemetry.tracer.exporter.batch.scheduledDelay=1000
mule.openTelemetry.tracer.exporter.batch.queueSize=2048
mule.openTelemetry.tracer.exporter.batch.maxSize=512
mule.openTelemetry.tracer.exporter.batch.backPressure.strategy=DROP
mule.openTelemetry.exporter.resource.service.name=hello-world-api
mule.openTelemetry.exporter.resource.service.namespace=test
```

**Production starting point — cost-aware, resilient:**

```
mule.openTelemetry.tracer.exporter.enabled=true
mule.openTelemetry.tracer.exporter.type=HTTP
mule.openTelemetry.tracer.exporter.endpoint=http://otel-collector-opentelemetry-collector.otel-collector.svc.cluster.local:4318/v1/traces
mule.openTelemetry.tracer.level=MONITORING
mule.openTelemetry.tracer.exporter.sampler=parentbased_traceidratio
mule.openTelemetry.tracer.exporter.sampler.arg=0.1
mule.openTelemetry.tracer.exporter.timeout=5s
mule.openTelemetry.tracer.exporter.compression=gzip
mule.openTelemetry.tracer.exporter.batch.scheduledDelay=5000
mule.openTelemetry.tracer.exporter.batch.queueSize=4096
mule.openTelemetry.tracer.exporter.batch.maxSize=512
mule.openTelemetry.tracer.exporter.batch.backPressure.strategy=DROP
mule.openTelemetry.tracer.exporter.backoff.maxAttempts=5
mule.openTelemetry.tracer.exporter.backoff.initial=1s
mule.openTelemetry.tracer.exporter.backoff.maximum=5s
mule.openTelemetry.tracer.exporter.backoff.multiplier=1.5
mule.openTelemetry.exporter.resource.service.name=hello-world-api
mule.openTelemetry.exporter.resource.service.namespace=payments-prod
```

---

## Summary

We've covered every Runtime Manager property available for the Direct Telemetry Stream traces feature. Here is the condensed guide:

**Control data flow.** `enabled`, `type`, `endpoint`, and `headers` define whether and where traces go.

**Control data depth.** `tracer.level` controls how many spans a single flow execution generates. Keep it at `MONITORING` unless we are actively debugging.

**Control data volume.** `sampler` and `sampler.arg` control what fraction of executions we trace at all. This is our most important cost lever in production.

**Control reliability.** The batch, backpressure, retry, timeout, and compression properties tune the mechanics of delivery. The defaults are sensible. We adjust them when volume grows or when we observe drops and errors in the Mule application logs.

**Control identity.** `service.name` and `service.namespace` make our traces findable in Splunk when many services share the same index.

In the next post in this series, we'll add OpenTelemetry **log export** alongside traces — and show how trace IDs and span IDs link our Splunk log events to the exact span where they were produced.

> ➡️ **Next up:** In Part 3 we'll cover log export configuration, linking spans to application logs, and building correlation queries in Splunk using trace IDs.

---

## References

- [MuleSoft Direct Telemetry Streams Documentation](https://docs.mulesoft.com/runtime-manager/direct-telemetry-streams)
- [OpenTelemetry Specification](https://opentelemetry.io/docs/reference/specification/)
- [OTLP Exporter Configuration](https://opentelemetry.io/docs/reference/specification/protocol/exporter/)
- [Splunk OTel Connector](https://docs.splunk.com/Observability/gdi/get-data-in/application/otel-connector.html)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
