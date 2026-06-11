---
title: "Distributed Traces from Mule via OpenTelemetry to OTel Collector and Elasticsearch — Part 7"
slug: "mule-distributed-traces-opentelemetry-collector-elasticsearch"
description: "Turn on Mule's Tracer Exporter, propagate W3C traceparent between two Mule 4.11 apps, route OTLP traces through the same OpenTelemetry Collector from Part 5, and land correlated spans alongside logs in Elasticsearch."
author: "Gonzalo Marcos"
date: 2026-06-09
status: not validated
lang: en
category: observability
tags:
  - mule-runtime
  - cloudhub-2
  - runtime-fabric
  - logging
  - architecture-diagram
  - english
  - series
  - deep-dive
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 7
type: tutorial
difficulty: advanced
read_time: 18
mule_version: "4.11"
platform:
  - anypoint-platform
  - cloudhub-2
  - runtime-fabric
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![CloudHub 2.0](https://img.shields.io/badge/Platform-CloudHub_2.0-00A0DF?logo=mulesoft&logoColor=white) ![Runtime Fabric](https://img.shields.io/badge/Platform-Runtime_Fabric-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Advanced](https://img.shields.io/badge/Level-Advanced-e74c3c) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 7](https://img.shields.io/badge/Part-7-16a085) ![18 min](https://img.shields.io/badge/Read_Time-18_min-lightgrey)

# Distributed Traces from Mule via OpenTelemetry to OTel Collector and Elasticsearch — Part 7

In [Part 5](#) we turned on Mule's **Direct Telemetry Stream — Logs Exporter** and stood up an OpenTelemetry Collector on a third EC2 instance. Logs from CloudHub 2.0 and Runtime Fabric have been flowing into the `mule-logs` index ever since. In [Part 6](#) we explained why OTLP/protobuf cannot be POSTed straight to Elasticsearch and why an OTel Collector — or a vendor-native OTLP receiver — has to sit in between. The collector is already there, idle on its `traces` pipeline, waiting for spans.

In this tutorial we will turn on Mule's **Tracer Exporter** with three Runtime Manager properties on each of **two** Mule 4.11 apps. App A's HTTP Requestor will call App B over HTTP, the W3C `traceparent` header will propagate across the boundary, and the OTel Collector will write the resulting two-span trace into the `mule-traces` index. Because we already set `mule.put.trace.id.and.span.id.in.mdc=true` in Part 5, every log line in `mule-logs` will arrive carrying the same `trace.id` as the spans in `mule-traces` — that is the correlation key Part 8's dashboard will join on.

This is the post where the series payoff lands: a single HTTP request becomes a multi-span trace plus correlated logs, and we can answer "what happened during *that* request?" with one Kibana query.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 6 — Why OpenTelemetry Cannot Ship Straight to Elastic — OTLP, Protobuf, and the Alternatives](#) · **Next:** Part 8 — Unified Kibana Dashboard: Logs + Traces Across Mule Apps

> [!WARNING]
> **HTTP-only, demo-grade.** OTLP traces leave the Mule runtime over plain HTTP, the OTel Collector exports to Elasticsearch over plain HTTP, and the credentials are Basic Auth. Acceptable inside a private VPC; not appropriate for shared networks. The Mule docs document mTLS for the production variant under `mule.openTelemetry.exporter.tls.*`.

> [!IMPORTANT]
> **Direct Telemetry Stream — Tracer Exporter requires:**
> - **Mule runtime 4.11.0** or later.
> - An **Advanced** or **Titanium** subscription tier.
> - **Anypoint Connector for HTTP 1.8** or later — this is the version that injects and reads the W3C `traceparent` header. Older HTTP Connector versions silently break trace context propagation, which makes a multi-app call look like two unrelated single-span traces.
>
> Source: [MuleSoft docs — OpenTelemetry support](https://docs.mulesoft.com/mule-runtime/latest/otel-support).

---

## What We Will Cover

- Build a second Mule 4.11 app (`hello-world-mule-downstream`) that the first app will call.
- Add a `traces` pipeline + a second `elasticsearch` exporter to the existing OTel Collector from Part 5.
- Turn on **Direct Telemetry Stream — Tracer Exporter** on both apps with the verbatim property names from the docs.
- Verify W3C `traceparent` propagation: one inbound request becomes a single trace with two spans, one per app.
- Confirm log↔trace correlation: every log line in `mule-logs` from this run carries the same `trace.id` as the spans in `mule-traces`.
- Build the muscle memory for trace-driven debugging — pick a slow request, follow its `trace.id` to every log line.

---

## Prerequisites

Before we start, we will need:

- The hello-world Mule project from [Part 5](#) deployed and shipping logs to `mule-logs` (call it **App A** for the rest of this post).
- A working **OTel Collector** on `10.0.1.30` from Part 5, currently routing logs into `mule-logs`.
- A working Elasticsearch + Kibana from [Parts 1–3](#), with the `mule-traces` index, `mule-traces-writer` role, and `mule-tracer` user from Part 3.
- An Anypoint Platform org with the **Advanced** or **Titanium** tier in the deployment environment.
- **Anypoint Connector for HTTP 1.8.x** or later, declared as a dependency in both apps' `pom.xml`.

We will sanity-check that the existing collector → Elastic leg still works for the traces user before changing anything:

```bash
# Synthetic OTLP/HTTP trace to confirm the collector accepts /v1/traces and ES accepts the traces user.
curl -X POST http://localhost:4318/v1/traces \
  -H 'Content-Type: application/json' \
  -d '{"resourceSpans":[{"scopeSpans":[{"spans":[{"traceId":"4bf92f3577b34da6a3ce929d0e0e4736","spanId":"00f067aa0ba902b7","name":"smoke-test","kind":1,"startTimeUnixNano":"'"$(date +%s)"'000000000","endTimeUnixNano":"'"$(date +%s)"'500000000"}]}]}]}'
```

```
curl -X POST http://127.0.0.1:4318/v1/traces \
  -H 'Content-Type: application/json' \
  -d '{"resourceSpans":[{"scopeSpans":[{"spans":[{"traceId":"4bf92f3577b34da6a3ce929d0e0e4736","spanId":"00f067aa0ba902b7","name":"smoke-test","kind":1,"startTimeUnixNano":"'"$(date +%s)"'000000000","endTimeUnixNano":"'"$(date +%s)"'500000000"}]}]}]}'
```


Then `curl` `mule-traces` directly:

```bash
curl -u mule-tracer:$MULE_TRACER_PASSWORD \
  "http://10.0.1.20:9200/mule-traces/_count?pretty"
```

We expect the count to bump. If it does not, fix the collector's traces pipeline before deploying any app.

---

## What Direct Telemetry Stream Buys Us for Traces

A trace is a tree of **spans**. Each span records *one unit of work* — receiving an HTTP request, calling a downstream service, executing a SQL query — with a start time, an end time, attributes, and a parent. A *distributed* trace is a tree that crosses process boundaries: the parent span lives in one Mule app, a child span lives in another.

What ties them together is the **W3C Trace Context**: a `traceparent` HTTP header that travels with the call. The header carries the trace ID (constant across the whole tree) and the parent span ID (the caller's span). The receiving app reads the header, makes its own span a child of the caller's span, and sends both spans to the same backend with the same trace ID.

Mule 4.11's Tracer Exporter implements OTel-spec span generation for every common Mule processor — HTTP Listener, Logger, Set Payload, HTTP Requestor, Database, DataWeave, Flow Reference. Anypoint Connector for HTTP 1.8+ injects and consumes the `traceparent` header automatically. The two pieces together mean: turn on the Tracer Exporter on each app and the HTTP boundary becomes invisible to the trace tree.

| Decision | Options Considered | Chosen | Rationale |
| --- | --- | --- | --- |
| Wire format | OTLP/gRPC vs OTLP/HTTP | OTLP/HTTP | Same endpoint shape as logs in Part 5; one collector receiver path for both signals; easier to debug with `curl` |
| Propagation | W3C Trace Context (`traceparent`) vs B3 (Zipkin) | W3C Trace Context | OTel default; HTTP Connector 1.8+ injects/reads it natively |
| Sampler | `alwaysOn` vs `traceidratio` 0.1 | `alwaysOn` for the demo | We want every request to produce a trace while we are wiring this up; flip to `traceidratio` for production |

---

## Overview

```
┌──────────────────────────────┐         ┌──────────────────────────────┐
│  App A — hello-world-mule    │ HTTP    │  App B — hello-world-        │
│  CH2 / RTF                   │  +      │  mule-downstream             │
│                              │ trace-  │  CH2 / RTF                   │
│  HTTP listener /hello        │ parent  │  HTTP listener /downstream   │
│  ├── Logger (parent span)    │────────▶│  ├── Logger (child span)     │
│  └── HTTP Requestor          │         │  └── Set Payload             │
│                              │         │                              │
│  Direct Telemetry Stream     │         │  Direct Telemetry Stream     │
│  Tracer Exporter ON          │         │  Tracer Exporter ON          │
│  Logs Exporter ON (Part 5)   │         │  Logs Exporter ON            │
└──────────────┬───────────────┘         └──────────────┬───────────────┘
               │ OTLP/HTTP                              │ OTLP/HTTP
               ▼                                        ▼
                 ┌────────────────────────────────────┐
                 │  OTel Collector — 10.0.1.30:4318   │
                 │   receivers: otlp                  │
                 │   exporters: elasticsearch         │
                 │   pipelines: logs / traces         │
                 └────────────────────────────────────┘
                            │ HTTP (Basic Auth)
                            ▼
                 ┌──────────────────────────────────┐
                 │  Elasticsearch — 10.0.1.20:9200  │
                 │   mule-logs   (mule-logger)      │
                 │   mule-traces (mule-tracer)      │
                 └──────────────────────────────────┘
```

---

## Table of Contents

1. [Step 1 — Build the Downstream Mule App](#step-1--build-the-downstream-mule-app)
2. [Step 2 — Wire the Upstream App's HTTP Requestor to Call the Downstream App](#step-2--wire-the-upstream-apps-http-requestor-to-call-the-downstream-app)
3. [Step 3 — Add the Traces Pipeline to the OTel Collector](#step-3--add-the-traces-pipeline-to-the-otel-collector)
4. [Step 4 — Turn On the Tracer Exporter on Both Apps](#step-4--turn-on-the-tracer-exporter-on-both-apps)
5. [Step 5 — Trigger an End-to-End Request](#step-5--trigger-an-end-to-end-request)
6. [Step 6 — Inspect the Trace and Its Correlated Logs in Kibana](#step-6--inspect-the-trace-and-its-correlated-logs-in-kibana)
7. [Verification](#verification)
8. [Troubleshooting](#troubleshooting)

---

### Step 1 — Build the Downstream Mule App

We will create a second Mule 4.11 project that mirrors App A's flow — HTTP Listener, Logger, Set Payload — but on a different path so we can call it from the first app.

In Anypoint Studio (or ACB):

1. **File → New → Mule Project**.
2. **Project Name:** `hello-world-mule-downstream`.
3. **Runtime:** Mule **4.11.x**.
4. **Group ID / Artifact ID:** match your org conventions; the artifact ID will be the default `service.name` if we forget to set it explicitly.

Paste this into `src/main/mule/downstream.xml`:

📄 `src/main/mule/downstream.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:http="http://www.mulesoft.org/schema/mule/http"
      xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="
        http://www.mulesoft.org/schema/mule/core
        http://www.mulesoft.org/schema/mule/core/current/mule.xsd
        http://www.mulesoft.org/schema/mule/http
        http://www.mulesoft.org/schema/mule/http/current/mule-http.xsd">

    <http:listener-config name="HTTP_Listener_config">
        <http:listener-connection host="0.0.0.0" port="8082"/>
    </http:listener-config>

    <flow name="downstream-flow">
        <http:listener config-ref="HTTP_Listener_config" path="/downstream"/>
        <logger level="INFO" message="Downstream - request received"/>
        <set-payload value='#[output application/json --- { "message": "Hello from downstream" }]'/>
    </flow>

</mule>
```

Confirm the HTTP Connector dependency in `pom.xml` is **1.8.0 or later**:

```xml
<dependency>
    <groupId>org.mule.connectors</groupId>
    <artifactId>mule-http-connector</artifactId>
    <version>1.8.0</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

> [!IMPORTANT]
> If we leave the dependency at an older 1.7.x version, the listener will not consume the incoming `traceparent` header. App B's span will still emit, but it will become the **root** of its own trace — not a child of App A's request. The Kibana view will then show two unrelated traces with different `trace.id`s. The fix is purely in `pom.xml`; nothing in the Mule XML changes.

Build and publish to Exchange:

```bash
mvn clean deploy -DskipTests
```

> [!NOTE]
> *Screenshot — Anypoint Exchange listing the `hello-world-mule-downstream` asset, version `1.0.0`.*

---

### Step 2 — Wire the Upstream App's HTTP Requestor to Call the Downstream App

We will edit App A from Part 5 (`hello-world-mule`) so the `/hello` flow makes a call to `hello-world-mule-downstream` before returning. This is the call that turns one request into two spans.

Update `src/main/mule/hello-world.xml` so the flow becomes:

📄 `src/main/mule/hello-world.xml`

```xml
<http:request-config name="Downstream_Request_config">
    <http:request-connection host="${downstream.host}" port="${downstream.port}"/>
</http:request-config>

<flow name="hello-world-flow">
    <http:listener config-ref="HTTP_Listener_config" path="/hello"/>
    <logger level="INFO" message="Hello World - request received"/>
    <http:request method="GET"
                  config-ref="Downstream_Request_config"
                  path="/downstream"
                  doc:name="Call Downstream"/>
    <logger level="INFO" message='#["Downstream replied: " ++ payload.message]'/>
    <set-payload value='#[output application/json --- { "message": "Hello World", "downstream": payload.message }]'/>
</flow>
```

The `${downstream.host}` and `${downstream.port}` placeholders are resolved at runtime from properties we will set in Step 4 — that lets us point App A at App B's CH2 hostname (or RTF ingress) without rebuilding.

Confirm App A's `pom.xml` also pins the HTTP Connector at **1.8.0 or later** for the *outbound* side:

```xml
<dependency>
    <groupId>org.mule.connectors</groupId>
    <artifactId>mule-http-connector</artifactId>
    <version>1.8.0</version>
    <classifier>mule-plugin</classifier>
</dependency>
```

Build and republish to Exchange:

```bash
mvn clean deploy -DskipTests
```

> [!NOTE]
> *Screenshot — Anypoint Studio canvas of `hello-world-flow` showing Listener → Logger → HTTP Requestor → Logger → Set Payload.*

---

### Step 3 — Add the Traces Pipeline to the OTel Collector

The collector from Part 5 already has a `logs` pipeline. We will add a `traces` pipeline that uses a second `elasticsearch` exporter — different user (`mule-tracer`), different index (`mule-traces`).

SSH into the collector host (`10.0.1.30`) and edit `/etc/otelcol-contrib/config.yaml`:

📄 `/etc/otelcol-contrib/config.yaml`

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    send_batch_size: 100
    timeout: 5s

exporters:
  elasticsearch/logs:
    endpoints: ["http://10.0.1.20:9200"]
    user: mule-logger
    password: ${env:MULE_LOGGER_PASSWORD}
    logs_index: mule-logs
    mapping:
      mode: raw

  elasticsearch/traces:
    endpoints: ["http://10.0.1.20:9200"]
    user: mule-tracer
    password: ${env:MULE_TRACER_PASSWORD}
    traces_index: mule-traces
    mapping:
      mode: raw

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch/logs]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch/traces]
```

📄 Full file: [./assets/otel-collector/config.yaml](./assets/otel-collector/config.yaml)

> [!IMPORTANT]
> Two `elasticsearch` exporters with the same type need **distinct names** — `elasticsearch/logs` and `elasticsearch/traces`. The `/<suffix>` syntax is the OTel Collector's convention for naming multiple instances of the same component. Without the suffix, the second one silently overrides the first and the logs pipeline stops working.

Add the `MULE_TRACER_PASSWORD` to the systemd drop-in we created in Part 5 Step 3.1:

```bash
sudo systemctl edit otelcol-contrib
```

Add a second `Environment=` line so the override now reads:

```ini
[Service]
Environment="MULE_LOGGER_PASSWORD=<MULE_LOGGER_PASSWORD>"
Environment="MULE_TRACER_PASSWORD=<MULE_TRACER_PASSWORD>"
```

> [!IMPORTANT]
> Each `Environment=` line takes one variable. **Do not** combine them on a single line — `Environment="A=1 B=2"` is parsed as a single variable named `A` with value `1 B=2`. One line per variable, every time.

Reload, restart, verify — the same recipe from Part 5 Step 3.3:

```bash
sudo systemctl daemon-reload
sudo systemctl restart otelcol-contrib
systemctl show otelcol-contrib -p Environment   # both vars should appear
sudo journalctl -u otelcol-contrib -f
```

Tail the journal until we see the `Everything is ready` line again. The collector is now serving both pipelines on the same OTLP receiver.

---

### Step 4 — Turn On the Tracer Exporter on Both Apps

Time for the runtime properties. Both apps get the same trace exporter properties — only the `service.name` differs.

**App A (`hello-world-mule-direct-stream-ch2` — pick whichever environment you have wired)**

In Runtime Manager → Applications → `<app>` → **Settings → Properties**, add:

| Property                                      | Value                                                  | Type     |
| --------------------------------------------- | ------------------------------------------------------ | -------- |
| `mule.openTelemetry.tracer.exporter.enabled`  | `true`                                                 | Property |
| `mule.openTelemetry.tracer.exporter.type`     | `HTTP`                                                 | Property |
| `mule.openTelemetry.tracer.exporter.endpoint` | `http://10.0.1.30:4318/v1/traces`                      | Property |
| `mule.openTelemetry.tracer.exporter.sampler`  | `alwaysOn`                                             | Property |
| `mule.openTelemetry.tracer.level`             | `MONITORING`                                           | Property |
| `downstream.host`                             | `hello-world-mule-downstream-ch2.<region>.cloudhub.io` | Property |
| `downstream.port`                             | `443`                                                  | Property |

> [!IMPORTANT]
> Property names mirror the logs exporter from Part 5, just with `tracer` instead of `logging`. **Casing is identical** — `openTelemetry` (camel-case `T`), `exporter` lowercase. The MuleSoft docs page lists `mule.openTelemetry.tracer.exporter.enabled`, `mule.openTelemetry.tracer.exporter.endpoint`, `mule.openTelemetry.tracer.exporter.type`, and `mule.openTelemetry.tracer.exporter.headers` verbatim.

> [!TIP]
> `mule.openTelemetry.tracer.exporter.sampler=alwaysOn` is what gets us a span on *every* request while we are debugging this. For production, switch to `traceidratio` and set `mule.openTelemetry.tracer.exporter.sampler.arg` to a fraction (e.g. `0.1` for 10% sampling). The default is `parentbased_alwaysOn` which respects the parent's sampling decision — fine once propagation is verified, but it can hide problems while we are wiring the apps up.

**App B (`hello-world-mule-downstream-ch2`)**

Same six properties, but **bump `service.name`**:

| Property | Value |
| --- | --- |
| `mule.openTelemetry.tracer.exporter.enabled` | `true` |
| `mule.openTelemetry.tracer.exporter.type` | `HTTP` |
| `mule.openTelemetry.tracer.exporter.endpoint` | `http://10.0.1.30:4318/v1/traces` |
| `mule.openTelemetry.tracer.exporter.sampler` | `alwaysOn` |
| `mule.openTelemetry.tracer.level` | `MONITORING` |
| `mule.openTelemetry.exporter.resource.service.name` | `hello-world-mule-downstream-ch2` |

> [!IMPORTANT]
> Apply the **same `mule.openTelemetry.logging.exporter.*` properties from Part 5 to App B** as well — the downstream app needs to ship logs into `mule-logs` for the `trace.id` correlation in Step 6 to work. If only App A ships logs, half the trace will have no log lines to join against.

Click **Apply Changes** on both apps. CH2 / RTF restarts each app once.

> [!NOTE]
> *Screenshot — Runtime Manager Properties tab on App A showing the Tracer Exporter rows side-by-side with the Logs Exporter rows from Part 5.*

---

### Step 5 — Trigger an End-to-End Request

Hit App A a few times so we have something to look at:

```bash
for i in 1 2 3; do
  curl -s "https://hello-world-mule-direct-stream-ch2.<region>.cloudhub.io/hello"
  echo
done
```

Each call should return:

```json
{"message":"Hello World","downstream":"Hello from downstream"}
```

Now grab the `trace.id` of one of those requests so we can chase it through Kibana. The simplest path: tail App A's logs in Runtime Manager (or in `mule-logs` directly) and copy the `trace.id` field of the most recent `Hello World - request received` line:

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_search?q=Body:%22Hello%20World%20-%20request%20received%22&size=1&sort=@timestamp:desc&pretty" \
  | jq -r '.hits.hits[0]._source | (.["Attributes.trace.id"] // .Attributes["trace.id"])'
```

We expect a 32-character hex string like `4bf92f3577b34da6a3ce929d0e0e4736`. We will use it in Step 6.

> [!TIP]
> If the field path looks awkward, that is the OTel Collector's `mapping.mode: raw` showing through — it preserves the OTLP field hierarchy verbatim. Part 8 will normalize this with a runtime field on the data view; for this post we read the raw shape directly.

---

### Step 6 — Inspect the Trace and Its Correlated Logs in Kibana

Two queries, one trace ID. Open Kibana → **Analytics → Discover**.

**Query 1 — All spans of this trace.**

Select the **Mule Traces** data view. Set a KQL filter:

```kql
TraceId : "4bf92f3577b34da6a3ce929d0e0e4736"
```

We expect **multiple documents per request** (typically one per Mule processor on each app's flow path) with these fields:

| Field | App A spans | App B spans |
| --- | --- | --- |
| `Resource.service.name` | `hello-world-mule-direct-stream-ch2` | `hello-world-mule-downstream-ch2` |
| `Name` | e.g. `mule:http:listener`, `mule:logger`, `mule:http:request` | e.g. `mule:http:listener`, `mule:set-payload` |
| `Kind` | `SPAN_KIND_SERVER` for the inbound listener; `SPAN_KIND_INTERNAL` for inner processors; `SPAN_KIND_CLIENT` for the outbound `http:request` to App B | `SPAN_KIND_SERVER` for the listener; `SPAN_KIND_INTERNAL` for inner processors |
| `ParentSpanId` | empty on App A's root span; populated on every other span | App B's listener span's `ParentSpanId` = App A's HTTP-request span's `SpanId` (this is what crosses the service boundary) |
| `Duration` | **microseconds** — `mule:set-payload` ~ 600 µs, full request a few ms | same units |
| `TraceStatus` | `0` unset, `1` OK, `2` ERROR | same |
| `location` | the Mule flow path, e.g. `main-flow/processors/2` | same |
| `correlation.id` | top-level dot-separated (note: logs use camelCase `correlationId`) | same |
| `artifact.id` / `artifact.type` | the Mule app's artifact ID and `app` | same |
| `thread.start.id` / `thread.start.name` / `thread.end.name` | Mule traces capture both span-start and span-end thread because async flows can switch threads mid-span | same |

The cross-service link: **App B's HTTP-listener span's `ParentSpanId` equals App A's HTTP-request span's `SpanId`.** That is the W3C `traceparent` propagation in action — no shared trace ID is enough; the parent-span linkage is what ties the two services into one trace tree.

> [!IMPORTANT]
> **`Duration` is in microseconds**, not nanoseconds. A `mule:set-payload` span typically reports a value in the hundreds (e.g. `651`); divide by `1000` to get milliseconds. The Part 8 dashboard's `duration_ms` runtime field uses this division.

> [!TIP]
> **Field-name asymmetry between logs and traces**: the same concept gets a different field on each index. Worth memorizing before writing dashboards:
> - **Correlation ID** — `correlationId` on logs, `correlation.id` on traces.
> - **Flow location** — `processorPath` on logs, `location` on traces.
> - **Thread** — `thread.id`/`thread.name` on logs, `thread.start.id`/`thread.start.name`/`thread.end.name` on traces.
>
> Traces capture *more* about Mule's threading model because spans cross thread boundaries; logs only see the thread the line was emitted on. Part 11's reshape unifies these where it makes sense (both → ECS `process.thread.*`).

**Query 2 — All logs of this trace.**

Switch to the **Mule Logs** data view. **Same KQL filter, same field name** — the OTel Collector's Elasticsearch exporter under `mapping.mode: raw` writes `TraceId` at the top level of `_source` for both signals:

```kql
TraceId : "4bf92f3577b34da6a3ce929d0e0e4736"
```

(Mule's MDC bridge from Part 5 also writes a parallel lowercase `trace_id` on every log record, so `trace_id : "..."` works equivalently. The two field names carry identical values; pick `TraceId` for canonical use because it is the OTLP-spec name.)

We expect **multiple documents** — at least one `Hello World - request received` from App A, one `Downstream replied: ...` from App A, one `Downstream - request received` from App B, plus any framework noise. Every one of them shares the same `TraceId`.

> [!NOTE]
> *Screenshot — Kibana Discover showing two spans in **Mule Traces** filtered by `TraceId`, then the same `TraceId` in **Mule Logs** returning the four log lines.*

> [!TIP]
> Kibana 8.x **Observability → APM** can often render OTel-shaped traces as a service map and a flame graph **if** the index is named `traces-apm-*` and conforms to the APM data stream schema. We deliberately sit outside that schema (we kept the `mule-traces` plain index from Part 3 to avoid coupling to Elastic APM). Part 8 builds an explicit dashboard with **Lens** that does not need APM at all.

---

## Verification

Three checks to confirm the trace path is wired end-to-end.

**1. Both apps emit spans, same `trace.id`.**

```bash
curl -u mule-tracer:$MULE_TRACER_PASSWORD \
  "http://10.0.1.20:9200/mule-traces/_search" \
  -H 'Content-Type: application/json' \
  -d '{
    "query": { "term": { "TraceId": "4bf92f3577b34da6a3ce929d0e0e4736" } },
    "size": 10
  }' | jq '.hits.hits[]._source | {service: .Resource["service.name"], parent: .ParentSpanId}'
```

We expect exactly two hits — one with empty `parent`, one with App A's `SpanId` as parent.

**2. Logs from both apps share the same `trace.id`.**

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_search?q=Attributes.trace.id:%224bf92f3577b34da6a3ce929d0e0e4736%22&size=20&pretty" \
  | jq -r '.hits.hits[]._source.Resource["service.name"]' | sort -u
```

We expect both `hello-world-mule-direct-stream-ch2` and `hello-world-mule-downstream-ch2` in the output. If only one shows up, App B's log exporter or trace-id MDC is missing — re-check Step 4.

**3. The collector counts both signals.**

The OTel Collector's Prometheus metrics on `:8888/metrics` expose both `otelcol_exporter_sent_log_records_total` and `otelcol_exporter_sent_spans_total`:

```bash
curl -s http://10.0.1.30:8888/metrics | grep -E '^otelcol_exporter_sent_(spans|log_records)_total'
```

Both should be increasing as we hit the endpoint.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Only App A produces spans; App B's spans are missing | Tracer Exporter not enabled on App B | Add the same six tracer properties to App B (Step 4) |
| Two apps produce spans but with **different** `trace.id` values | HTTP Connector below 1.8 on either app — `traceparent` header not propagated | Bump `mule-http-connector` to 1.8.0+ in both `pom.xml`s, redeploy |
| App A produces a span; App B receives the request but its span has empty `ParentSpanId` (root) | Listener side of the HTTP Connector is below 1.8, or a non-HTTP boundary (JMS, Anypoint MQ) is between them without context propagation | Bump connector versions; for non-HTTP transports use the connector-specific propagation extension |
| Spans arrive but they all have `Kind: SPAN_KIND_INTERNAL` | `mule.openTelemetry.tracer.level=DEBUG` is producing internal spans only | Set `mule.openTelemetry.tracer.level=MONITORING` (the default for production tracing in Mule) |
| `mule-logs` documents do not have `TraceId` (or `trace_id`) | OTLP-spec trace context absent on the OTLP record — typically because the Logger fired outside an active span (startup, scheduler tick, async error handler), or the line was emitted before the trace context was set on the thread | The trace context is set automatically when an HTTP listener or other span-instrumented operation handles the request; lines emitted on those threads carry it. For lines emitted outside a span (e.g. boot logs), there is genuinely no trace to bind. `mule.put.trace.id.and.span.id.in.mdc=true` only adds the **lowercase `trace_id`/`span_id`** MDC duplicates of the already-set OTLP-spec values; it cannot fabricate a trace where none exists. |
| Trace context appears twice on every record (`TraceId` and `trace_id`, `SpanId` and `span_id`) | The OTel SDK writes the OTLP-spec capitalized fields; the MDC bridge **also** writes lowercase underscore-separated copies | Both are correct — pick `TraceId`/`SpanId` as canonical for queries (OTLP-spec). To deduplicate, drop the lowercase ones in the OTel Collector with a `transform` rule (covered in Part 11) |
| Collector journal shows `403 Forbidden` from Elastic on the traces pipeline | `mule-tracer` user does not have `write` on `mule-traces` | Re-check Part 3 Step 2 — `mule-traces-writer` role with `read`, `write`, `auto_configure` |
| Collector journal shows `failed to start "elasticsearch" exporter: duplicate component` | Two exporters named just `elasticsearch` | Use the suffixed names `elasticsearch/logs` and `elasticsearch/traces` (Step 3) |
| `MULE_TRACER_PASSWORD` resolves empty in collector logs | Drop-in not picked up | Walk through the four checks from Part 5 Step 3.4; remember `daemon-reload` + `restart`, in that order |
| Trace exists in `mule-traces` but Discover shows zero rows | Data view has the wrong time field | In **Stack Management → Data Views → Mule Traces**, set time field to `@timestamp` (or whichever the exporter version uses) |
| Sampler dropped 90% of demo traffic | `mule.openTelemetry.tracer.exporter.sampler` is at the default ratio sampler in some setups | Set `sampler=alwaysOn` while debugging; flip to `traceidratio` + `sampler.arg=0.1` for production |
| **Traces land in `mule-traces` but logs do not in `mule-logs`, no app errors** | Logs are reaching the collector but Elasticsearch is rejecting them on parse — almost always a dynamic-mapping conflict against a previous (Part 4) writer's shape | Add a `debug` exporter to the **logs** pipeline to bisect Mule-side vs ES-side: <br/>1. Edit `/etc/otelcol-contrib/config.yaml`: add `debug: { verbosity: detailed }` to `exporters:` and `[elasticsearch/logs, debug]` to `service.pipelines.logs.exporters`. <br/>2. `sudo /usr/bin/otelcol-contrib validate --config=/etc/otelcol-contrib/config.yaml && sudo systemctl restart otelcol-contrib`. <br/>3. Hit `/hello`, then `sudo journalctl -u otelcol-contrib -f`. <br/> If `debug` prints a `LogsExporter` block with the log record → Mule is sending; look further down the journal for `failed to index document` from `elasticsearch/logs` (next row). <br/> If `debug` is silent → Mule is not sending; check `mule.openTelemetry.logging.exporter.level=INFO` *and* the app's Log4j root level — the OTel exporter taps Log4j's in-process event stream, so a root level above INFO emits nothing. <br/>Cross-check with `curl -s http://localhost:8888/metrics \| grep otelcol_receiver_accepted_log_records` — non-zero means at least one record reached the receiver. |
| **Collector journal shows `failed to index document ... document_parsing_exception ... failed to parse field [thread] of type [text]`** | `mule-logs` was first written by Part 4's Log4j2 `JsonLayout`, which mapped `thread` as a `text` string. OTel's logs exporter writes `thread.id` and `thread.name` as a nested object — type collision, ES rejects every batch. Same root cause for `Resource`, `process`, and any other field whose first writer wrote a string but the second writes an object. | Pick one strategy: <br/>**(1) Drop and recreate `mule-logs` so the OTel shape claims the dynamic mapping fresh** (cheap if the index only holds demo data): `curl -X DELETE -u elastic:$ELASTIC_PASSWORD "http://10.0.1.20:9200/mule-logs" && curl -X PUT -u elastic:$ELASTIC_PASSWORD "http://10.0.1.20:9200/mule-logs"`. The `mule-logs-writer` role's `auto_configure` privilege then lets the exporter shape the new mapping on first write. <br/>**(2) Use separate indexes per writer** — point Part 4's HTTP appender at `mule-logs-log4j` and the OTel exporter at `mule-logs-otlp`. Update the data view to `mule-logs-*`. <br/>**(3) Force ECS field names on both sides** — switch the `elasticsearch/logs` exporter to `mapping.mode: ecs` and migrate Part 4's `log4j2.xml` to `JsonTemplateLayout` with `eventTemplateUri="classpath:EcsLayout.json"`. Both writers then emit the same field names and the same index works. <br/>**Remember to remove the `debug` exporter** from `service.pipelines.logs.exporters` once the issue is resolved — `verbosity: detailed` dumps every record to the journal and fills the disk fast. |

---

## What We Covered

- We built **`hello-world-mule-downstream`** — a second Mule 4.11 app — and updated App A to call it via the HTTP Requestor.
- We added a **`traces` pipeline** and a second `elasticsearch/traces` exporter to the OTel Collector from Part 5, sharing the same OTLP receiver.
- We turned on **Direct Telemetry Stream — Tracer Exporter** on both apps with the verbatim property names from the MuleSoft docs.
- We confirmed **W3C `traceparent` propagation**: a single inbound request becomes a single trace with two spans, parent and child, across two services.
- We confirmed **log↔trace correlation**: every `mule-logs` document from both apps carries the same `Attributes.trace.id` as the spans, because Part 5's `mule.put.trace.id.and.span.id.in.mdc=true` is doing its job.

We now have logs and traces in the same cluster, correlated by `trace.id`, queryable separately or together. The next post turns this into a single screen.

> ➡️ **Next up:** In Part 8 we will assemble a **unified Kibana dashboard** that joins `mule-logs` and `mule-traces` on `trace.id`. Panels: log volume by `service.name`, error rate per service, slow-trace table with click-through to the matching log lines, a span-duration histogram by service. We will export the dashboard NDJSON and ship it as a repo asset so any future cluster can import it in one click. That post closes the series.

---

## References

- [OpenTelemetry support for Mule runtime — MuleSoft Docs](https://docs.mulesoft.com/mule-runtime/latest/otel-support)
- [Direct Telemetry Stream for Traces Configuration — MuleSoft Docs](https://docs.mulesoft.com/mule-runtime/latest/otel-support#direct-telemetry-stream-for-traces-configuration)
- [Anypoint Connector for HTTP — Release Notes](https://docs.mulesoft.com/release-notes/connector/connector-http)
- [W3C Trace Context — `traceparent` Specification](https://www.w3.org/TR/trace-context/)
- [OpenTelemetry — Sampling](https://opentelemetry.io/docs/concepts/sampling/)
- [OpenTelemetry Collector — Pipelines](https://opentelemetry.io/docs/collector/configuration/#pipelines)
- [Elasticsearch Exporter — OTel Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/elasticsearchexporter)
