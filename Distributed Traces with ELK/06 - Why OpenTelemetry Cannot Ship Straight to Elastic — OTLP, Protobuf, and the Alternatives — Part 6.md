---
title: "Why OpenTelemetry Cannot Ship Straight to Elastic — OTLP, Protobuf, and the Alternatives — Part 6"
slug: "otel-otlp-protobuf-vs-json-alternatives-elastic-mule"
description: "A conceptual deep-dive into what OpenTelemetry support in Mule actually means, why OTLP/protobuf cannot be sent directly to Elasticsearch's _doc endpoint, and the full set of alternatives — Log4j2, Filebeat, Logstash, Elastic APM Server, OTel Collector, native OTLP."
author: "Gonzalo Marcos"
date: 2026-06-08
status: not validated
lang: en
category: observability
tags:
  - mule-runtime
  - logging
  - elasticsearch
  - architecture-diagram
  - best-practices
  - english
  - series
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 6
type: deep-dive
difficulty: intermediate
read_time: 17
mule_version: "4.11"
platform:
  - anypoint-platform
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Deep_Dive](https://img.shields.io/badge/Type-Deep_Dive-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 6](https://img.shields.io/badge/Part-6-16a085) ![17 min](https://img.shields.io/badge/Read_Time-17_min-lightgrey)

# Why OpenTelemetry Cannot Ship Straight to Elastic — OTLP, Protobuf, and the Alternatives — Part 6

In [Part 5](#) we turned on Mule's **Direct Telemetry Stream** for logs and pointed the runtime at an OpenTelemetry Collector running on its own EC2 instance. A reasonable question came up while we were configuring it: *why do we need the collector at all? Why can't we set `mule.openTelemetry.logging.exporter.endpoint` to `http://<elastic>:9200/mule-logs/_doc` and an `Authorization: Basic ...` header, the way the Log4j2 appender does in Part 4?*

The short answer is: **the wire formats don't match.** Mule's exporter emits **OTLP** with **protobuf** payloads; Elasticsearch's `/_doc` endpoint expects **JSON**. The longer answer involves what OpenTelemetry actually is, what "OTel support in Mule" really means, and what the full alternative space looks like — including paths that don't use OpenTelemetry at all (Log4j2 over HTTP, Filebeat, Logstash) and paths that use OpenTelemetry but skip the collector (Elastic APM Server, Elastic Cloud's native OTLP endpoint).

This post is a deep-dive, not a tutorial. We will not deploy anything. We will end with a clear decision matrix that should make every "where do my logs go?" question for the rest of this series — and the next one — answerable in five seconds.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 5 — Hello-World Mule App on CloudHub 2.0 / RTF: Direct Telemetry Stream for Logs](#) · **Next:** Part 7 — Distributed Traces from Mule via OpenTelemetry → OTel Collector → Elasticsearch

---

## What We Will Cover

- What "OpenTelemetry support in Mule runtime 4.11+" actually buys us.
- What **OTLP** is — the OpenTelemetry Protocol — and what's inside an OTLP request.
- What **Protocol Buffers (protobuf)** are, how they differ from JSON, and why an HTTP endpoint that expects JSON cannot consume protobuf.
- The seven realistic ways to externalize Mule logs and traces, grouped into "uses OTel" and "does not use OTel."
- Where **Logstash** fits — and why it is closer to Filebeat in this story than to the OTel Collector.
- A decision table to pick the right path for a given environment, subscription tier, and operational appetite.

---

## What "OpenTelemetry Support in Mule" Means

OpenTelemetry (OTel) is a CNCF project that defines:

1. **A data model** for the three observability signals — **traces**, **logs**, and **metrics**.
2. **APIs and SDKs** in every major language for emitting those signals.
3. **A wire protocol — OTLP — for sending the signals to a backend.**
4. **A reference receiver — the OpenTelemetry Collector — that ingests OTLP and re-emits it in any backend's format.**

When MuleSoft says "Mule runtime 4.11 supports OpenTelemetry," they mean exactly piece #3: **the runtime itself implements the OTLP exporter for logs and traces** (and metrics from 4.12). It does *not* mean the runtime understands every backend's API. It means: feed me a property tree, and I will emit OTLP at the endpoint you tell me.

That is a deliberate, sensible scope. OTel's whole value proposition is *"emit once in a standard format; let an intermediary translate to whichever backend(s) you care about."* The Mule runtime is the producer. Translating OTLP into a backend's native API is somebody else's job.

> [!IMPORTANT]
> **OpenTelemetry support in Mule = OTLP exporter implementation.** It is not a magic "send my logs to any vendor" button. The runtime can talk to anything that speaks OTLP. Anything that does not speak OTLP needs a translator in front of it.

---

## What OTLP Actually Is

OTLP — the **OpenTelemetry Protocol** — is the wire format Mule uses when Direct Telemetry Stream is on. The spec defines two transport variants:

| Variant | Default port | Endpoint paths | Payload format |
| --- | --- | --- | --- |
| **OTLP/gRPC** | `4317` | n/a (gRPC method calls) | Protocol Buffers (protobuf) binary |
| **OTLP/HTTP** | `4318` | `/v1/traces`, `/v1/logs`, `/v1/metrics` | Protocol Buffers binary (default) **or** JSON-encoded protobuf (optional) |

Two things to note:

1. The **default** payload for both gRPC and HTTP is **binary protobuf**, not JSON. Mule's Direct Telemetry Stream uses the default — binary.
2. OTLP/HTTP has an *optional* JSON encoding mode in the spec. It still uses the same protobuf message schemas — the JSON is just a different on-the-wire serialization. Mule does not currently expose a property to switch to JSON encoding, so on Mule we always send binary.

That is the entire OTLP "shape." A POST against `http://<host>:4318/v1/logs` with a binary protobuf body and a `Content-Type: application/x-protobuf` header. There is **no `_doc` semantic, no Basic Auth scheme, no `Authorization: Basic` header support, no automatic index routing.** Those are Elasticsearch concepts, and OTLP knows nothing about them.

---

## Protobuf vs JSON — Why They Are Not Interchangeable

Both protobuf and JSON serialize structured data, but they are not drop-in replacements for each other.

### JSON in one paragraph

JSON is **self-describing text**. A JSON object carries its field names *with the data*, every time:

```text
{"@timestamp":"2026-06-08T10:14:23Z","level":"INFO","message":"hello"}
```

The `@timestamp`, `level`, and `message` strings appear literally in the bytes on the wire. The receiver does not need any prior agreement to parse the document — it can look at the curly braces, the colons, the quotes, and figure out the structure character by character. That is what makes JSON nice to debug with `curl` and what makes Elasticsearch's `/_doc` API trivial to use: send any JSON, ES indexes it.

### Protobuf in one paragraph

Protocol Buffers ("protobuf") is **a binary format with an out-of-band schema**. Both sender and receiver must already know the *same* `.proto` file — a definition like:

```protobuf
message LogRecord {
  fixed64 time_unix_nano = 1;
  string severity_text   = 2;
  AnyValue body          = 3;
}
```

On the wire, **field names are gone.** Each field becomes a tiny tag (`1`, `2`, `3`) plus the encoded value, packed tightly. A 200-byte JSON log line might be 60 bytes of protobuf. The cost is that the receiver cannot make sense of the bytes without the matching `.proto`.

### Why HTTP `_doc` cannot consume OTLP

When the OTel Collector — or any OTLP-aware receiver — gets a POST at `/v1/logs`, it:

1. Reads `Content-Type: application/x-protobuf`.
2. Loads the OTLP `.proto` schemas (compiled in at build time).
3. Decodes the binary body field-by-field into typed objects.
4. Hands the structured `LogRecord`s to whatever pipeline comes next.

Elasticsearch's `/_doc` endpoint:

1. Reads `Content-Type: application/json`.
2. Calls a JSON parser on the body.
3. Indexes whatever fields the JSON has.

If you point Mule at `http://<es>:9200/mule-logs/_doc`, what arrives is `Content-Type: application/x-protobuf` and a binary body. ES tries to JSON-parse it, sees a non-`{` first byte, and returns a `400 mapper_parsing_exception` — or worse, indexes garbage strings if the bytes happen to look text-like. ES has no `.proto` for OTLP, no logic to flatten OTLP `Resource` / `Scope` / `LogRecord` into ECS fields, no understanding of `severityNumber` vs `severityText`. The two systems speak different languages.

> [!IMPORTANT]
> **The data is the same; the language is not.** A log record carries the same information whether it is JSON or protobuf — timestamp, severity, body, attributes. What differs is *how those bytes are serialized*. You cannot consume one as the other without translating.

| Property | JSON | OTLP / Protobuf |
| --- | --- | --- |
| Self-describing | Yes — field names in every message | No — needs the `.proto` schema |
| Human-readable on the wire | Yes (debuggable with `curl`) | No (binary, even with HTTP transport) |
| Size on the wire | Larger | ~30–50% smaller, typically |
| Schema evolution | Loose — add a field, old readers ignore it | Strict — field tags survive renames; deletions need care |
| Authentication shape | Whatever the consuming HTTP API requires (Basic, API key, OAuth) | Defined by OTLP spec (`headers` config carries arbitrary HTTP headers in OTLP/HTTP) |
| Receiver complexity | Any HTTP server | Anything that has compiled the OTLP `.proto` |

---

## The Seven Alternatives, Grouped

Now to the practical question: with OTLP being the only thing Mule's Direct Telemetry Stream emits, what can actually receive it — and what other paths exist if we don't use Direct Telemetry Stream at all?

### Group A — Does **not** use OpenTelemetry

#### A1. Log4j2 HTTP appender (Part 4 of this series)

Mule app's `log4j2.xml` defines an `<Http>` appender that POSTs JSON straight to `http://<es>:9200/<index>/_doc`. No OTLP, no collector.

- **Pros:** No subscription tier requirement, no extra components, full control of JSON shape, works on every Mule target including standalone.
- **Cons:** Per-app config, per-event HTTP request (no batching by default), credentials to manage in `log4j2.xml`/JVM args/Secure Properties.
- **Where it lives in this series:** Part 4.

#### A2. Filebeat sidecar (or DaemonSet)

A Filebeat agent runs *next to* the Mule runtime, tails the on-disk log files (the default Mule log4j2 RollingFile output), and POSTs to Elasticsearch's bulk API. The Mule app is unchanged — it just writes to disk as it always has.

- **Pros:** Decouples log shipping from the app entirely. Native Elastic component, ships with batching, retries, ECS templates, and ILM-friendly index strategies. Works unmodified on standalone and RTF (DaemonSet pattern).
- **Cons:** Not a viable path on **CloudHub 2.0** — we don't own the host filesystem. Requires a place to drop the agent (sidecar or DaemonSet on RTF/k8s; systemd unit on standalone). One more component to operate.
- **Where it could fit:** A bonus post in this series, or the natural answer for any standalone/RTF deployment that wants to skip both Log4j HTTP and OTel.

#### A3. Logstash with the HTTP input

Logstash runs as a service, exposing an HTTP input that accepts JSON. Mule's Log4j2 `<Http>` appender posts to Logstash instead of directly to Elastic; Logstash applies filters, enrichment, and ECS conversion, then writes to Elasticsearch.

- **Pros:** Heavy ETL — `grok`, GeoIP, mutate, drop, conditional routing — all pre-Elastic. Same Mule-side wire-up as Part 4, just a different URL.
- **Cons:** A whole JVM to operate (Logstash is heavier than the OTel Collector or Filebeat). For pure forwarding we don't need it. Most of what Logstash does for free, Elastic ingest pipelines now also do — natively, inside Elasticsearch.
- **Where it fits:** A teams-with-existing-Logstash story. **Not recommended as a new install** for a 2026 greenfield deployment unless you already have Logstash filter logic you want to keep.

> [!TIP]
> **Logstash is in this list, but it is closer to Filebeat than to the OTel Collector.** Both Logstash and Filebeat are *Elastic* tools; both speak Elastic-flavored JSON. Neither natively understands OTLP/protobuf. There *is* a Logstash OTLP input plugin (community-maintained) but it is a translator from OTel to Logstash's pipeline — exactly what the OTel Collector already does, but with more memory.

### Group B — Uses OpenTelemetry

#### B1. Direct Telemetry Stream → OTel Collector → Elasticsearch (Part 5 of this series)

Mule emits OTLP/HTTP; the OpenTelemetry Collector receives it, runs an `elasticsearch` exporter, writes JSON to ES.

- **Pros:** Standard OTLP wire format, the same collector handles logs *and* traces (and metrics in 4.12+), zero `log4j2.xml` changes, runtime-level config in Runtime Manager properties, easy to fan out to additional backends later (Splunk, Datadog, S3).
- **Cons:** Requires Mule **4.11+** and **Advanced/Titanium** Anypoint subscription. Adds an EC2 instance/DaemonSet to operate. Anypoint Monitoring's own OTel exporter becomes unavailable while Direct Telemetry Stream is on.
- **Where it lives in this series:** Part 5 (logs) and Part 7 (traces).

#### B2. Direct Telemetry Stream → Elastic APM Server → Elasticsearch

APM Server is Elastic's own component that **natively speaks OTLP** (gRPC and HTTP) and writes to Elasticsearch's APM data streams. Mule properties point straight at APM Server, no OTel Collector in the picture.

- **Pros:** Single Elastic-managed binary instead of OTel Collector + exporter config. Tighter integration with Kibana's **Observability → APM** UI, the **Service Map**, latency distributions, error rates. APM API keys are first-class auth.
- **Cons:** Couples us to Elastic's APM data streams (`traces-apm-*`, `logs-apm.app-*`) instead of our `mule-logs` / `mule-traces` indexes — that is the *point*, but it does mean the Part 3 indexes/users/roles are bypassed for the APM-indexed data. Another component to operate. Vendor-specific by design.
- **Where it could fit:** A natural successor series — *"From OTel Collector to Elastic APM."*

#### B3. Direct Telemetry Stream → Elastic Cloud / Serverless native OTLP endpoint

Elastic Cloud Serverless and recent Elastic Cloud Hosted projects expose **native OTLP endpoints** (`/v1/traces`, `/v1/logs`, `/v1/metrics`) that ingest directly. Mule properties point at the cloud URL with an API-key header, and we're done.

- **Pros:** Zero infrastructure on our side. Mule properties are the only thing we configure.
- **Cons:** **Self-managed Elastic 8.x on EC2 — what we built in Part 1 — does not ship this.** Only Elastic Cloud Serverless and recent Elastic Cloud Hosted projects expose it. Vendor lock-in to Elastic Cloud. Cost model is consumption-based.
- **Where it could fit:** A "the cloud version of this series" reference — out of scope here.

#### B4. Anypoint Monitoring's built-in Telemetry Exporter (the one we *turned off*)

Anypoint Monitoring has its own OTel exporter that ships logs and traces to *its* pipeline. Customers can then forward to a configured backend without touching the Mule runtime properties at all.

- **Pros:** No collector to deploy, no Mule properties to set per-app, the destination is configured in the Anypoint UI.
- **Cons:** **Mutually exclusive with Direct Telemetry Stream** — turning Direct Stream on disables this. Couples us to Anypoint's pipeline (if it's degraded, we lose data). The set of supported destinations is a curated list, not "any OTLP receiver."
- **Where it fits:** Orgs that want to stay inside the Anypoint UI and don't need the flexibility of running their own collector. We deliberately picked Direct Telemetry Stream in Part 5 *over* this one, but it is a real alternative.

---

## The Decision Matrix

| Path | Need Mule 4.11+? | Need Advanced/Titanium tier? | Extra component to operate | Wire format Mule emits | Works on CH2 / RTF / Standalone | Fine-grained JSON control |
| --- | --- | --- | --- | --- | --- | --- |
| **A1. Log4j2 HTTP appender** | No (any 4.x) | No | None | JSON | All three | Full |
| **A2. Filebeat sidecar/DaemonSet** | No | No | Filebeat | JSON (Filebeat's) | Standalone, RTF | Limited (Filebeat module) |
| **A3. Logstash HTTP input** | No | No | Logstash + Log4j2 appender to Logstash URL | JSON | All three | Full (Logstash filters) |
| **B1. Direct Stream → OTel Collector → ES** *(Part 5/7)* | **Yes** | **Yes** | OTel Collector | OTLP/HTTP (protobuf) | CH2, RTF, Standalone | Collector exporter config |
| **B2. Direct Stream → Elastic APM Server → ES** | **Yes** | **Yes** | APM Server | OTLP/HTTP | CH2, RTF, Standalone | APM data streams (no choice) |
| **B3. Direct Stream → Elastic Cloud native OTLP** | **Yes** | **Yes** | None (vendor-managed) | OTLP/HTTP | CH2, RTF, Standalone | Limited |
| **B4. Anypoint Monitoring Telemetry Exporter** | No (4.5+ I think) | **Yes** (Anypoint Monitoring) | None | OTLP (handled by Anypoint) | CH2, RTF | None |

How to read this:

- **No subscription budget for Advanced/Titanium?** A1 (Log4j2) is your only realistic option for "logs to ES from CH2." On standalone/RTF, A2 (Filebeat) is the operational best practice if you can run an agent.
- **Already on Advanced/Titanium and want to go all-in on OTel?** B1 is the right default for self-managed Elastic; B2 if you live in Kibana's APM UI; B3 if you are on Elastic Cloud Serverless.
- **Have a Logstash investment?** A3 lets you keep it without touching Mule beyond a URL change.
- **Just want the Anypoint UI to be the source of truth?** B4. Trade flexibility for fewer moving parts.

---

## Common Misconceptions

> [!WARNING]
> **"OTel means I can point Mule at any endpoint."** No — it means Mule emits a *standard* format. The receiver still has to speak that format. Pointing OTLP at Elasticsearch's `/_doc` does not work for the same reason that emailing a PDF to a fax machine does not work.

> [!WARNING]
> **"OTLP/HTTP" sounds like JSON because it's HTTP.** The default payload of OTLP/HTTP is **binary protobuf**, not JSON. The HTTP transport is just a TCP-friendly way to deliver protobuf bytes. There is an optional JSON-encoded protobuf mode in the spec, but Mule does not expose a switch for it.

> [!WARNING]
> **"Logstash is the OTel Collector for Elastic."** Logstash predates OTel by a decade. It ingests Beats, syslog, JSON over HTTP, files, and a long list of other inputs — all in Elastic-flavored shapes. The OTel Collector ingests OTLP and a list of vendor formats, processes them, and exports to a different list of vendor formats. There is overlap (both can write to ES, both can transform), but they are not the same kind of tool.

> [!WARNING]
> **"Direct Telemetry Stream is just a config flag — it should work with any URL."** It is a config flag, but the code behind the flag is an OTLP exporter. Pointing it at a non-OTLP URL fails silently or noisily depending on the receiver — the runtime keeps trying, the destination keeps rejecting, no log shows up.

---

## Where This Leaves the Series

The choice we made for the rest of this series — **OTel Collector in front of Elasticsearch** — is path **B1**. We picked it because:

- It is the path the MuleSoft docs explicitly recommend for backends that do not natively ingest OTLP.
- It is the only path that lets us add a second backend (a future Splunk, Datadog, or S3 archive) by changing one collector config, not every Mule app.
- The same collector handles logs (Part 5) **and** traces (Part 7) **and** metrics (when we move to Mule 4.12), which is the point of OpenTelemetry in the first place.
- The Elastic APM Server path (B2) is a strong alternative — and the natural follow-up series.

The Log4j2 path from Part 4 is not retired; it is still the right answer for Mule versions older than 4.11, for organizations on the base Anypoint subscription, and for environments where another component is one component too many.

> ➡️ **Next up:** In Part 7 we will turn on Mule's **Tracer Exporter** with `mule.openTelemetry.tracer.exporter.*` and add a `traces` pipeline to the same OTel Collector we built in Part 5. Two Mule apps will propagate `traceparent` to each other so a request becomes a multi-span trace, and because we already turned on `mule.put.trace.id.and.span.id.in.mdc=true` in Part 5, every log line in `mule-logs` will already carry the matching `trace.id`. That is what unlocks the unified dashboard in Part 8.

---

## References

- [OpenTelemetry support for Mule runtime — MuleSoft Docs](https://docs.mulesoft.com/mule-runtime/latest/otel-support)
- [OTLP Specification — OpenTelemetry](https://opentelemetry.io/docs/specs/otlp/)
- [OTLP/HTTP Encoding — OpenTelemetry Spec](https://opentelemetry.io/docs/specs/otlp/#otlphttp)
- [Protocol Buffers — Language Guide (proto3)](https://protobuf.dev/programming-guides/proto3/)
- [Protocol Buffers vs. JSON — Encoding Comparison](https://protobuf.dev/programming-guides/encoding/)
- [OpenTelemetry Collector Architecture](https://opentelemetry.io/docs/collector/architecture/)
- [Elasticsearch Exporter — OTel Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/elasticsearchexporter)
- [Elastic APM Server — OTLP Native Ingest](https://www.elastic.co/guide/en/observability/current/apm-open-telemetry-direct.html)
- [Filebeat Reference — Elastic Docs](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
- [Logstash HTTP Input Plugin — Elastic Docs](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-http.html)
- [Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/index.html)
