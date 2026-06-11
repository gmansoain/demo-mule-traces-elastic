---
title: "Standardizing the Mule Log Schema — Path B — Centralizing on the OTel Collector — Part 10"
slug: "mule-log-schema-otel-collector-resource-attributes-processors"
description: "Implement the same Mule log schema as Part 9 — but centrally on the OTel Collector — using resource, attributes, and routing processors. One YAML, every app, no per-app rebuild."
author: "Gonzalo Marcos"
date: 2026-06-09
status: not validated
lang: en
category: observability
tags:
  - mule-runtime
  - logging
  - architecture-diagram
  - best-practices
  - english
  - series
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 10
type: tutorial
difficulty: advanced
read_time: 16
mule_version: "4.11"
platform:
  - anypoint-platform
  - cloudhub-2
  - runtime-fabric
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![CloudHub 2.0](https://img.shields.io/badge/Platform-CloudHub_2.0-00A0DF?logo=mulesoft&logoColor=white) ![Runtime Fabric](https://img.shields.io/badge/Platform-Runtime_Fabric-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Advanced](https://img.shields.io/badge/Level-Advanced-e74c3c) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 10](https://img.shields.io/badge/Part-10-16a085) ![16 min](https://img.shields.io/badge/Read_Time-16_min-lightgrey)

# Standardizing the Mule Log Schema — Path B — Centralizing on the OTel Collector — Part 10

In [Part 9](#) we adopted an eleven-field log schema and implemented it entirely on the Mule side: deployment-static fields via `mule.openTelemetry.exporter.resource.attributes`, per-request fields via Log4j MDC. Path A is the right answer when the team that owns the Mule app also owns the schema — but it scales linearly with the number of apps. Two apps means two Properties tabs to keep in sync. Twenty apps means twenty places where a schema rename has to land, twenty Maven builds to bump, twenty deployments to push.

This post is the alternative implementation. Path B keeps the **same schema** from Part 9, but defines it **once, on the OTel Collector**, in a single YAML file the platform team owns. Mule apps stay clean — they emit OTLP with whatever defaults they have, and the collector enriches every record with the standardized fields before it reaches Elasticsearch. The cost is operational: the collector now carries schema logic, and a misconfigured processor can break every Mule app's logs at once.

We will reuse the field catalogue from Part 9 verbatim. The only change is *where the wiring lives*.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 9 — Standardizing the Mule Log Schema — Path A — Mule-Side Resource Attributes and MDC](#) · **Next:** Part 11 — Reshaping OTLP Logs into ECS-Native JSON for Elastic — OTel Collector Processors as ETL

> [!WARNING]
> **HTTP-only, demo-grade.** Same posture as the rest of the series. Centralizing schema on the collector also centralizes blast radius — a typo in the YAML can corrupt every Mule app's logs in one restart. Run the collector under change-managed config (Git, code review) once you go past one team.

> [!IMPORTANT]
> **You do not need to undo Part 9 to follow this post.** Path A and Path B compose: per-deployment values like `service.name` belong on Path A (the Mule app knows what it is); cross-cutting values like `business.domain` or `data.classification` can move to Path B. The trade-off table at the end gives a clean recipe for splitting the schema between the two.

---

## What We Will Cover

- The collector-side equivalent of every group from Part 9: deployment-static, per-request, per-event, trace context.
- Three OTel Collector processors that do the heavy lifting: **`resource`**, **`attributes`**, and **`transform`**.
- How to **route per `service.name`** so different Mule apps get different schema decorations from the same collector.
- A consolidated `config.yaml` that adds the Part 9 schema entirely on the collector, with the Mule apps stripped back to a minimal property set.
- Verification — every log document carries the same eleven fields as in Part 9, and `mule-traces` documents do too.
- A trade-offs table that gives a clear answer for *when* to use Path A, Path B, or a hybrid.

---

## Prerequisites

- A working OTel Collector from [Part 5](#), with the systemd drop-in and the two `elasticsearch/{logs,traces}` exporters.
- The `mule-logs` and `mule-traces` indexes from [Part 3](#) and the dashboard from [Part 8](#).
- A Mule 4.11 app shipping logs and traces via Direct Telemetry Stream (Part 5 / Part 7).
- The collector binary version `v0.110.0` or later — the `transform` processor's OTTL syntax in earlier versions is incompatible with the snippets below.

We will sanity-check the current collector version before changing anything:

```bash
otelcol-contrib --version
```

We expect `otelcol-contrib version v0.110.0` or higher.

---

## A Quick Detour — OTLP Layer vs. Elasticsearch `_source` Layer

Before we write any OTTL, we need to be honest about where things live. The OTel Collector's processors operate on the **OTLP record on the wire**, not on the JSON document Elasticsearch eventually indexes. There are two layers, and they look different.

| Layer | What it has | Example: where the MDC key `client_id` lives |
| --- | --- | --- |
| **OTLP record** (in the collector) | Strict OTLP shape: `Resource.attributes`, `LogRecord.attributes`, `LogRecord.body`, `LogRecord.severity_text`, `LogRecord.scope` | `LogRecord.attributes["client_id"]` |
| **Elasticsearch `_source`** (after `mapping.mode: raw` exporter) | The exporter writes each LogRecord attribute as a top-level key of `_source` — but **dots in the attribute name expand into nested JSON objects**. `Resource.*` keeps the same expansion under the `Resource` object. `Body` / `SeverityText` / `Scope` are top-level. | `_source.client_id` (flat, peer of `Body`) — but a key called `http.route` lands at `_source.http.route` (nested) |

That mismatch confuses people the first time they write OTTL against a production document. **OTTL paths use the OTLP layer.** When we write `set(attributes["foo"], "bar")` we are setting an attribute on the LogRecord — and **that attribute will appear at top level of `_source`** in `mapping.mode: raw`, not inside an `attributes` object. There is no `_source.attributes` field in the indexed document despite OTTL using `attributes[...]` in the syntax.

> [!IMPORTANT]
> **Dots in attribute names get expanded into nested objects.** Mule pushes the MDC key `http.route` (with a dot) — Elasticsearch ingests it as `{ "http": { "route": "/hello" } }`, *not* `"http.route": "/hello"`. The same applies on `Resource`: the property `deployment.environment=prod` lands as `Resource.deployment.environment` in dotted notation but as `Resource: { "deployment": { "environment": "prod" } }` in actual document structure. Discover's field selector hides this — it shows the dotted path. The raw `_source` JSON is where it bites.

A real `_source` document from CloudHub 2.0 / RTF **logs** after Part 9's properties are applied (verified, banking-scenario example):

```json
{
  "@timestamp": "2026-06-10T11:57:49.891Z",
  "Body": "{ \"message\": \"Hello Gon!\" }",
  "SeverityText": "INFO",
  "SeverityNumber": 9,
  "TraceFlags": 1,
  "TraceId": "6b3dd1cc097c3106aab24f73c257b429",
  "SpanId": "8aeb0eb3f121b7d7",
  "Resource": {
    "api":         { "layer": "process" },
    "business":    { "domain": "payments" },
    "deployment":  { "environment": "prod", "target": "runtime-fabric" },
    "mule":        { "runtime": { "version": "4.12.0" } },
    "service":     {
      "name": "process-payments-sepa",
      "namespace": "retail-banking",
      "version": "1.2.0",
      "instance": { "id": "2494d5ed-..." }
    },
    "envId": "09584395-...", "orgId": "98b7f48b-...", "rootOrgId": "37aa8fe0-...",
    "workerId": "64d8569b7d-wrk79",
    "telemetry": { "sdk": { "name": "opentelemetry", "language": "java", "version": "1.60.1" } }
  },
  "Scope": { "name": "org.mule.runtime.core.internal.processor.LoggerMessageProcessor", "version": "" },
  "correlationId":  "9a8231a0-...",
  "processorPath":  "hello-gon-subflow/processors/4",
  "client_id":      "Mobile App",
  "http_method":    "GET",
  "http":           { "route": "/hello" },
  "thread":         { "id": 29, "name": "[MuleRuntime].uber.04: ..." },
  "trace_id":       "6b3dd1cc097c3106aab24f73c257b429",
  "span_id":        "8aeb0eb3f121b7d7",
  "trace_flags":    "01"
}
```

Notice the two different shapes within the per-event attributes:

- **Flat top-level**: `correlationId`, `processorPath`, `client_id`, `http_method`, `trace_id`, `span_id`, `trace_flags`, `Body`, `SeverityText`. Mule pushes these MDC keys without dots, so they stay flat.
- **Nested**: `http.route` becomes `http: { route: "/hello" }`. `thread.id` / `thread.name` end up inside a single `thread` object. Same expansion rule as Resource.

In OTTL we still address them as `attributes["correlationId"]`, `attributes["client_id"]`, `attributes["http.route"]` — because at the OTLP layer the dotted name is just a single attribute key. The expansion happens in the exporter on the way out.

A real `_source` document from **traces** (same trace, different span, banking scenario):

```json
{
  "@timestamp":    "2026-06-10T11:57:49.890Z",
  "EndTimestamp":  "2026-06-10T11:57:49.891Z",
  "Name":          "mule:set-payload",
  "Kind":          "SPAN_KIND_INTERNAL",
  "Duration":      734,
  "TraceStatus":   0,
  "TraceId":       "6b3dd1cc097c3106aab24f73c257b429",
  "SpanId":        "3afb0a4a96af4998",
  "ParentSpanId":  "2579b300ea87f146",
  "Link":          "[]",
  "Resource":      { /* identical Resource block to logs — same Part 9 properties */ },
  "Scope":         { "name": "mule", "version": "1.0.0" },
  "artifact":      { "id": "mule-ds-logs", "type": "app" },
  "correlation":   { "id": "9a8231a0-..." },
  "location":      "hello-gon-subflow/processors/5",
  "client_id":     "Mobile App",
  "http_method":   "GET",
  "http":          { "route": "/hello" },
  "thread":        {
    "start": { "id": "29", "name": "[MuleRuntime].uber.04: ..." },
    "end":   { "name": "[MuleRuntime].uber.04: ..." }
  }
}
```

Three asymmetries between **logs** and **traces** survive even after Part 9. The other asymmetries you may have read about in older drafts of this post are gone — once Path A's `service.name` override is applied, both signals carry the same `Resource.service.name`.

| Concept | Logs (`_source`) | Traces (`_source`) |
| --- | --- | --- |
| Correlation ID | `correlationId` (top-level, camelCase, **no expansion** because no dot) | `correlation.id` → `correlation: { id: "..." }` (Mule emits the key with a dot, the exporter nests it) |
| Flow location | `processorPath` (top-level, camelCase) | `location` (top-level, different name entirely) |
| Mule artifact identity | not present | `artifact.id` + `artifact.type` → `artifact: { id, type }` (auto-injected by Mule's tracer for spans only) |
| Thread capture | `thread.id` / `thread.name` → `thread: { id, name }` (single emitting thread) | `thread.start.id` / `thread.start.name` / `thread.end.name` → `thread: { start: {...}, end: {...} }` (Mule captures both endpoints because async flows switch threads) |
| Duration | n/a | `Duration` (top-level, **microseconds** — typical `mule:set-payload` lands ~700 µs) |
| Status | `SeverityText` (top-level, string) | `TraceStatus` (top-level, numeric: `0` unset, `1` OK, `2` ERROR) |
| Span timestamps | `@timestamp` only | `@timestamp` (start) + `EndTimestamp` (end) |

The OTTL we write below has to handle this asymmetry — the same processor cannot apply to both signal types unchanged. Either write separate processors per signal (and apply them to the right pipeline) or write a single processor with `where` guards for each signal's actual shape.

> [!IMPORTANT]
> **OTTL does not see Elasticsearch `_source`. It sees the OTLP record.** When you write a `transform` rule, ask "what would this look like in OTLP?" — not "what would this look like in Discover?" If your OTTL works against the wrong layer, the rule silently does nothing.

> [!TIP]
> **Why some attribute keys have dots and others have underscores.** It is purely the name we (or Mule) chose at the *push* site. The OTel Collector and the Elasticsearch exporter neither rename nor flatten — they just carry the key through. `client_id` is flat in `_source` because Mule's tracing module wrote `client_id` (underscore). `http.route` is nested because Mule wrote `http.route` (dot). If we want a flat `client_id` and a flat `http_route`, we change the variable name on the Mule side. If we want everything nested and ECS-shaped (`process.thread.name`, `client.id` → `labels.client_id`, etc.) we wait for Part 11's reshape — that is exactly what it does.

---

## How OTel Collector Processors Map to the Part 9 Taxonomy

The processors do not replace the taxonomy from Part 9 — they are the collector-side implementation of each group.

| Part 9 group | Collector processor | What it does | Example |
| --- | --- | --- | --- |
| Deployment-static | `resource` | Adds, renames, or deletes attributes on the **Resource** of every signal | Add `deployment.environment=prod`, `api.layer=process` |
| Per-request | `attributes` | Adds, renames, or deletes attributes on the **LogRecord** / span | Rename incoming `processorPath` → `flow.location` |
| Per-event renames | `transform` (OTTL) | Field-level conditional transformations using a small expression language | If `Body` matches `^Inbound`, set `event.kind=inbound_request` |
| Per-app-different | `routing` (connector) | Routes signals into different pipelines based on a field, then applies a different processor chain to each route | App `process-payments-sepa` → `business.domain=payments` ; app `system-corebank-accounts` → `business.domain=accounts` |
| Drop unwanted | `filter` | Drops records or attributes matching a condition | Drop `mule-logs` records where `SeverityText IN ['DEBUG','TRACE']` in `prod` |

Three of these — `resource`, `attributes`, and `transform` — cover almost every standardization need we have. We will use all three in this post; `routing` and `filter` get a callout and a recipe but are not part of the main flow.

> [!TIP]
> **OTTL** (OpenTelemetry Transformation Language) is the small expression language behind the `transform` processor. It is *not* general-purpose — it is intentionally limited to safe, declarative ops on telemetry data. Read the spec link in the **References** section if you find yourself reaching for `if/else` or string regex; chances are OTTL has a one-liner for the case.

---

## The Schema, Re-anchored on the Collector

The eleven fields from Part 9, with their new home in the collector config:

| Field | Group | Path A (Part 9) | Path B (this post) |
| --- | --- | --- | --- |
| `service.name` | Deployment-static | `mule.openTelemetry.exporter.resource.service.name=...` | **Stays on Mule** — the app must identify itself |
| `service.namespace` | Deployment-static | `mule.openTelemetry.exporter.resource.service.namespace=...` | Stays on Mule (or moved to `routing` table) |
| `service.version` | Deployment-static | `service.version=1.2.0` in `resource.attributes` | Stays on Mule (lifecycle bound to the build) |
| `service.instance.id` | Infrastructure-derived | Auto-filled by runtime | Auto-filled by runtime — never touch |
| `deployment.environment` | Deployment-static | `deployment.environment=prod` in `resource.attributes` | **Moves to collector** — `resource` processor |
| `deployment.target` | Deployment-static | `deployment.target=runtime-fabric` in `resource.attributes` | Moves to collector |
| `deployment.region` | Deployment-static | (not in Part 9 baseline) | Moves to collector — easy to derive from EC2 metadata |
| `mule.runtime.version` | Deployment-static | `mule.runtime.version=4.11.0` in `resource.attributes` | Moves to collector |
| `api.layer` | Domain | `api.layer=process` in `resource.attributes` | Moves to collector via `routing` (per-app value) |
| `business.domain` | Domain | `business.domain=payments` in `resource.attributes` | Moves to collector via `routing` |
| `correlationId` (logs) / `correlation.id` (traces) | Per-request | Mule MDC default | **Stays on Mule** — Mule writes both names automatically; we cannot move them to the collector without losing the request-scoped value |
| `client_id` | Per-request | MDC via the Tracing module subflow (Part 9 Step 3) | Stays on Mule — value is lifted from the inbound `client_id` header |
| `http.route`, `http_method` | Per-request | MDC via subflow | Stays on Mule |
| `trace_id` / `TraceId`, `span_id` / `SpanId` | Trace context | OTLP-spec auto-injection + Mule's `mule.put.trace.id.and.span.id.in.mdc=true` MDC bridge | Stays on Mule |

The pattern: **Mule keeps fields it knows the value of *at runtime per request*. The collector takes over fields whose value is a function of *which app is calling it* and *what environment it is running in*.** That is the dividing line that makes the hybrid sensible.

---

## Overview — Path B

```
┌────────────────────────────┐
│  Mule app (Mule 4.11)      │
│                            │
│  Properties (minimal):     │
│   service.name=...         │
│   service.namespace=...    │
│   service.version=...      │
│                            │
│  MDC (per-request):        │
│   client_id, http.route,   │
│   http_method, correlationId,│
│   trace_id / TraceId       │
└────────────┬───────────────┘
             │ OTLP/HTTP — minimal Resource, full Attributes
             ▼
┌─────────────────────────────────────────────────────────────────┐
│  OTel Collector — schema enrichment lives here                  │
│                                                                 │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ processors.resource:                                   │    │
│   │   add deployment.environment, deployment.target,       │    │
│   │   deployment.region, mule.runtime.version              │    │
│   └────────────────────────────────────────────────────────┘    │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ processors.routing (by Resource.service.name):         │    │
│   │   process-payments-sepa  → set api.layer=process,      │    │
│   │                                business.domain=payments│    │
│   │   process-payments-swift → set api.layer=process,      │    │
│   │                                business.domain=payments│    │
│   │   system-corebank-accounts → set api.layer=system,     │    │
│   │                                business.domain=accounts│    │
│   └────────────────────────────────────────────────────────┘    │
│   ┌────────────────────────────────────────────────────────┐    │
│   │ processors.transform:                                  │    │
│   │   conditional event.outcome from Body / SeverityText   │    │
│   └────────────────────────────────────────────────────────┘    │
│                                                                 │
└────────────────────────────────────┬────────────────────────────┘
                                     │ HTTP + Basic Auth
                                     ▼
                          ┌──────────────────────────┐
                          │  Elasticsearch           │
                          │   mule-logs / mule-traces│
                          └──────────────────────────┘
```

---

## Table of Contents

1. [Step 1 — Strip the Path A Properties Off the Mule Apps](#step-1--strip-the-path-a-properties-off-the-mule-apps)
2. [Step 2 — Add the `resource` Processor for Universal Static Fields](#step-2--add-the-resource-processor-for-universal-static-fields)
3. [Step 3 — Add the `routing` Connector for Per-App Decorations](#step-3--add-the-routing-connector-for-per-app-decorations)
4. [Step 4 — Add the `transform` Processor for Conditional Enrichment](#step-4--add-the-transform-processor-for-conditional-enrichment)
5. [Step 5 — Wire Both Pipelines and Restart](#step-5--wire-both-pipelines-and-restart)
6. [Step 6 — Verify in Kibana](#step-6--verify-in-kibana)
7. [Verification](#verification)
8. [Troubleshooting](#troubleshooting)
9. [Trade-offs — When Path A, When Path B, When Both](#trade-offs--when-path-a-when-path-b-when-both)

---

### Step 1 — Strip the Path A Properties Off the Mule Apps

If we adopted Part 9 first, the Mule app's Properties tab carries eight `resource.*` properties. For Path B we keep only the three that are intrinsically per-app:

| Property | Keep / Remove | Why |
| --- | --- | --- |
| `mule.openTelemetry.exporter.resource.service.name` | **Keep** | The app must identify itself; the collector cannot guess |
| `mule.openTelemetry.exporter.resource.service.namespace` | **Keep** | Logical group above `service.name`; binds to the team, not the env |
| `mule.openTelemetry.exporter.resource.attributes` (with `service.version=...`) | **Keep** | Bound to the build artifact, lives in `pom.xml`, not the cluster |
| `mule.openTelemetry.exporter.resource.attributes` (everything else: `deployment.environment`, `deployment.target`, `api.layer`, `business.domain`, `mule.runtime.version`) | **Remove** | Now owned by the collector |
| `mule.openTelemetry.logging.exporter.*` | **Keep** | Direct Telemetry Stream wiring, not schema |
| `mule.openTelemetry.tracer.exporter.*` | **Keep** | Same |
| `mule.put.trace.id.and.span.id.in.mdc=true` | **Keep** | MDC bridge, not schema |

Apply Changes; the app restarts once.

> [!TIP]
> Keeping `service.name` and `service.version` on Mule is non-negotiable on Path B. The next step's `routing` processor matches on `service.name` to decide which decorations to apply — if Mule does not declare it, the collector cannot route.

---

### Step 2 — Add the `resource` Processor for Universal Static Fields

These four fields apply to **every** Mule app: same value regardless of `service.name`. They go in a single `resource` processor.

Edit `/etc/otelcol-contrib/config.yaml`:

```yaml
processors:
  batch:
    send_batch_size: 100
    timeout: 5s

  resource/static:
    attributes:
      - key: deployment.environment
        value: prod
        action: insert
      - key: deployment.target
        value: runtime-fabric
        action: insert
      - key: deployment.region
        value: eu-central-1
        action: insert
      - key: mule.runtime.version
        value: "4.11.0"
        action: insert
```

The `action: insert` only sets the value if the field is **not already present**. That is what allows Path A and Path B to coexist — if the Mule app already declared `deployment.environment`, the collector leaves it alone. Switch to `action: upsert` if you want the collector's value to *override* the app's, or to `action: update` if you want to update only when the field is already present.

> [!IMPORTANT]
> `action: insert` is almost always the right choice for the static processor. `upsert` looks tempting (*"the platform always wins"*) but it makes the Mule property a lie — you set it on the app, you do not see it in Elastic, and the next debugger blames the app instead of the collector. If the app's value is wrong, **fix the app**, do not paper over it from the collector.

> [!TIP]
> The processor name `resource/static` uses OTel Collector's named-instance syntax — the suffix after `/` lets us have multiple `resource` processors with different roles. We will add a `resource/per-app` later via the `routing` step. Without the suffix, the second `resource` processor silently overrides the first.

---

### Step 3 — Add the `routing` Connector for Per-App Decorations

Some fields are static per-app but vary across apps: `business.domain` is `payments` for `process-payments-sepa` and `process-payments-swift`, but `accounts` for `system-corebank-accounts`. Likewise, `api.layer` is `process` for the two payments APIs and `system` for the core-bank integration. The OTel Collector solves this with the **`routing` connector**, which fans signals into separate pipelines based on a field value, and lets each pipeline apply its own processor chain.

```yaml
connectors:
  routing/by-service:
    default_pipelines: [logs/default]
    error_mode: ignore
    table:
      - statement: route() where resource.attributes["service.name"] == "process-payments-sepa"
        pipelines: [logs/payments-sepa]
      - statement: route() where resource.attributes["service.name"] == "process-payments-swift"
        pipelines: [logs/payments-swift]
      - statement: route() where resource.attributes["service.name"] == "system-corebank-accounts"
        pipelines: [logs/corebank-accounts]

  routing/by-service-traces:
    default_pipelines: [traces/default]
    error_mode: ignore
    table:
      - statement: route() where resource.attributes["service.name"] == "process-payments-sepa"
        pipelines: [traces/payments-sepa]
      - statement: route() where resource.attributes["service.name"] == "process-payments-swift"
        pipelines: [traces/payments-swift]
      - statement: route() where resource.attributes["service.name"] == "system-corebank-accounts"
        pipelines: [traces/corebank-accounts]
```

And the per-app resource processors:

```yaml
processors:
  # ... resource/static, batch as before ...

  resource/payments-sepa:
    attributes:
      - key: api.layer
        value: process
        action: insert
      - key: business.domain
        value: payments
        action: insert
      - key: payments.rail
        value: sepa
        action: insert

  resource/payments-swift:
    attributes:
      - key: api.layer
        value: process
        action: insert
      - key: business.domain
        value: payments
        action: insert
      - key: payments.rail
        value: swift
        action: insert

  resource/corebank-accounts:
    attributes:
      - key: api.layer
        value: system
        action: insert
      - key: business.domain
        value: accounts
        action: insert
```

The `routing` connector is the cleanest way to apply *different* processors per app. The alternative — branching with `transform` — works but reads worse and gets messy fast.

> [!IMPORTANT]
> The `routing` *connector* (not the older `routing` *processor*) is the supported mechanism from `v0.96` onward. Older blog posts reference `processors.routing`, which is deprecated. If the YAML linter rejects the snippets above, confirm the collector version with `otelcol-contrib --version` and confirm the connector path with `otelcol-contrib components | grep routing`.

> [!TIP]
> If the per-app table is more than a handful of rows, consider lifting it out of `config.yaml` into an external file. The `${file:...}` resolver lets a config reference another file's contents — we can keep the per-app routing table in `routing-table.yaml` and let the schema team review *only* that file.

---

### Step 4 — Add the `transform` Processor for Conditional Enrichment

Some fields cannot be set unconditionally — they depend on the *content* of the log record. `event.outcome` is the canonical example: `success` for INFO records, `failure` for ERROR or FATAL. We will use the `transform` processor with OTTL.

```yaml
processors:
  # ... earlier processors ...

  transform/event-outcome:
    error_mode: ignore
    log_statements:
      - context: log
        statements:
          - set(attributes["event.outcome"], "success") where severity_text == "INFO"
          - set(attributes["event.outcome"], "success") where severity_text == "DEBUG"
          - set(attributes["event.outcome"], "failure") where severity_text == "WARN"
          - set(attributes["event.outcome"], "failure") where severity_text == "ERROR"
          - set(attributes["event.outcome"], "failure") where severity_text == "FATAL"
```

Now every `mule-logs` record has `Attributes.event.outcome`, derived from its severity, with no Mule change. Future panels can compute success rate as `count(event.outcome=success) / count()` instead of having to enumerate severity values.

> [!TIP]
> `transform` is also where you would build derived fields based on the log message — e.g. `set(attributes["event.kind"], "inbound_request") where IsMatch(body, "^Inbound")`. Be careful with regex on every record at high volume; the cheaper alternative is to log the structured field on the Mule side via MDC.

---

### Step 5 — Wire Both Pipelines and Restart

The full `service.pipelines` block tying receivers, processors, the routing connector, and the exporters together:

```yaml
service:
  pipelines:
    # Step 1: receive OTLP, attach the universal static fields,
    #         then fan out by service.name
    logs/in:
      receivers: [otlp]
      processors: [resource/static, transform/event-outcome, batch]
      exporters: [routing/by-service]

    # Per-app branches — each adds its own resource decorations and exports
    logs/payments-sepa:
      receivers: [routing/by-service]
      processors: [resource/payments-sepa]
      exporters: [elasticsearch/logs]

    logs/payments-swift:
      receivers: [routing/by-service]
      processors: [resource/payments-swift]
      exporters: [elasticsearch/logs]

    logs/corebank-accounts:
      receivers: [routing/by-service]
      processors: [resource/corebank-accounts]
      exporters: [elasticsearch/logs]

    # Default branch — for apps not in the routing table
    logs/default:
      receivers: [routing/by-service]
      processors: []
      exporters: [elasticsearch/logs]

    # Mirror the same shape for traces
    traces/in:
      receivers: [otlp]
      processors: [resource/static, batch]
      exporters: [routing/by-service-traces]

    traces/payments-sepa:
      receivers: [routing/by-service-traces]
      processors: [resource/payments-sepa]
      exporters: [elasticsearch/traces]

    traces/payments-swift:
      receivers: [routing/by-service-traces]
      processors: [resource/payments-swift]
      exporters: [elasticsearch/traces]

    traces/corebank-accounts:
      receivers: [routing/by-service-traces]
      processors: [resource/corebank-accounts]
      exporters: [elasticsearch/traces]

    traces/default:
      receivers: [routing/by-service-traces]
      processors: []
      exporters: [elasticsearch/traces]
```

📄 Full file: [./assets/otel-collector/config-path-b.yaml](./assets/otel-collector/config-path-b.yaml)

Validate, restart, watch:

```bash
sudo /usr/bin/otelcol-contrib validate --config=/etc/otelcol-contrib/config.yaml
sudo systemctl restart otelcol-contrib
sudo journalctl -u otelcol-contrib -f
```

We expect `Everything is ready` and an empty error log. The `validate` command catches every typo before it bites a real request — run it before every restart.

> [!IMPORTANT]
> Notice that `transform/event-outcome` runs in the `logs/in` pipeline, **before** the `routing` fan-out. Why: the transform applies to *every* log regardless of `service.name`, so it makes no sense to repeat it per-app. Putting it before the routing also means a typo in one of the per-app processors does not block the cross-cutting transform. Order the processors by *who-applies-to-everyone* first, then *who-routes*, then *who-decorates-per-route*.

---

### Step 6 — Verify in Kibana

Hit the app:

```bash
curl -s -H "x-client-id: ios-app-3.4" \
  "https://hello-world-mule-direct-stream-rtf.<rtf-ingress>/products"
```

In Kibana → **Discover** → **Mule Logs** → most recent document. The `_source` shape we actually get (verified against a real CH2 / RTF document) — **MDC keys at top level, not under `Attributes`**:

```json
{
  "@timestamp": "2026-06-10T11:57:49.891Z",
  "Body": "Inbound request received",
  "SeverityText": "INFO",
  "SeverityNumber": 9,
  "TraceFlags": 1,
  "TraceId": "6b3dd1cc097c3106aab24f73c257b429",
  "SpanId": "8aeb0eb3f121b7d7",
  "Resource": {
    "service":     {                          // from Mule properties (Part 9)
      "name": "process-payments-sepa",
      "namespace": "retail-banking",
      "version": "1.2.0",
      "instance": { "id": "2494d5ed-..." }
    },
    "deployment":  { "environment": "prod", "target": "runtime-fabric", "region": "eu-central-1" },  // env/target from Mule, region from collector resource/static
    "mule":        { "runtime": { "version": "4.12.0" } },                                            // from Mule
    "api":         { "layer": "process" },                                                            // from collector resource/payments-sepa
    "business":    { "domain": "payments" },                                                          // from collector resource/payments-sepa
    "payments":    { "rail": "sepa" },                                                                // from collector resource/payments-sepa
    "envId": "09584395-...", "orgId": "98b7f48b-...",                                                 // from Anypoint runtime auto-fill
    "rootOrgId": "37aa8fe0-...", "workerId": "64d8569b7d-wrk79",
    "telemetry":   { "sdk": { "name": "opentelemetry", "language": "java", "version": "1.60.1" } }
  },
  "Scope": {
    "name": "org.mule.runtime.core.internal.processor.LoggerMessageProcessor",
    "version": ""
  },
  "correlationId":  "9a8231a0-...",          // from Mule MDC (auto) — TOP LEVEL
  "processorPath":  "hello-gon-subflow/processors/4",
  "client_id":      "Mobile App",            // from Mule MDC subflow — TOP LEVEL (no dot, stays flat)
  "http_method":    "GET",                   // from Mule MDC subflow — TOP LEVEL
  "http":           { "route": "/hello" },   // from Mule MDC subflow — NESTED because key has a dot
  "thread":         { "id": 29, "name": "[MuleRuntime].uber.04: ..." },
  "trace_id":       "6b3dd1cc097c3106aab24f73c257b429",
  "span_id":        "8aeb0eb3f121b7d7",
  "trace_flags":    "01",
  "event":          { "outcome": "success" } // from collector transform — NESTED because we declared it as event.outcome
}
```

Two observations:

1. **`event.outcome` is a new field** that did not exist in Part 9. That is the kind of derivation Path B unlocks: a rule that touches every app's logs, added in one YAML file, no app rebuild.
2. **Notice the four Anypoint Resource fields** (`envId`, `orgId`, `rootOrgId`, `workerId`) injected by the runtime regardless of our config. Part 9's catalogue covers them; do not redeclare.

---

## Verification

**1. Static fields land on every record.**

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_search" \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "bool": {
        "must_not": [{ "exists": { "field": "Resource.deployment.environment" } }]
      }
    },
    "size": 1
  }'
```

We expect `hits.total.value == 0`. Any record without `Resource.deployment.environment` is one the static processor missed — typo, ordering, or `action: update` instead of `insert`.

**2. Per-app decorations match the routing table.**

```bash
for svc in process-payments-sepa process-payments-swift system-corebank-accounts; do
  echo "=== $svc ==="
  curl -s -u mule-logger:$MULE_LOGGER_PASSWORD \
    "http://10.0.1.20:9200/mule-logs/_search?size=1&q=Resource.service.name:%22$svc%22" \
    | jq '.hits.hits[0]._source.Resource | { service: .["service.name"], domain: .["business.domain"], layer: .["api.layer"] }'
done
```

We expect each service to carry its own `business.domain` and `api.layer` from the per-app processor.

**3. The `event.outcome` transform fires.**

Force an error path on the Mule app (or temporarily change the Logger to ERROR). Confirm:

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_search?q=SeverityText:ERROR&size=1" \
  | jq '.hits.hits[0]._source.Attributes["event.outcome"]'
```

`"failure"`. Same query against `SeverityText:INFO` returns `"success"`.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Static fields missing on some records | `action: update` instead of `insert` on a record where the field did not exist | Use `action: insert`; `update` only writes if the key already exists |
| Routing connector drops records | `routing/by-service` has no `default_pipelines:` and a `service.name` does not match any rule | Add `default_pipelines: [logs/default]` (and `traces/default`) so unmatched records still flow |
| Records routed to the wrong app | `service.name` typo on the Mule side | `Resource.service.name` is the source of truth; fix the Mule property, not the routing table |
| `transform` rule sets `event.outcome` to literal string `unknown` | OTTL silently casts unknown attributes to strings; the `severity_text` matcher uses the OTLP enum, not the indexed text | OTTL's `severity_text` resolves correctly in `v0.110+`; if on older versions, use `severity_number == SEVERITY_NUMBER_INFO` instead |
| Old `processors.routing` config breaks after collector upgrade | The processor was deprecated in favor of the **connector** | Move it to `connectors.routing/...` and reference under both `receivers:` (downstream pipeline) and `exporters:` (upstream pipeline) |
| `validate` passes but the collector errors at startup | Some processors only validate at runtime (e.g. unreachable referenced pipeline IDs) | Read `journalctl` immediately after restart; the error names the missing pipeline ID |
| Schema applied to logs but not traces | Forgot to add `routing/by-service-traces` and the trace pipelines | Mirror the logs configuration for traces; the `resource` processors can be reused on both signal types |
| Per-app branch fails for one app while others work | YAML indentation drift in the per-app processor block | Run `yamllint` on `config.yaml` before every commit |

---

## Trade-offs — When Path A, When Path B, When Both

| Concern | Path A (Part 9 — Mule-side) | Path B (this post — Collector-side) |
| --- | --- | --- |
| **Where the schema lives** | Each Mule app's Runtime Manager Properties | One YAML on the OTel Collector |
| **Who owns the schema** | The team that owns the Mule app | The platform / observability team |
| **Cost of a schema rename** | N redeploys (one per app) | One collector restart |
| **Cost of a typo** | Affects one app | Affects every app at once |
| **Per-app conditional logic** | Native (it is the app's own properties) | Requires `routing` connector + per-app processors |
| **Coupling to runtime version** | None | Collector version pinned to OTTL syntax |
| **Visibility in Anypoint UI** | Properties tab shows the values | Anypoint UI sees only what Mule emits — opaque to the platform value-add |
| **Testability of a schema change** | Redeploy the app to a sandbox | Redeploy the collector config to a sandbox collector |
| **Failure mode** | One app's schema breaks; others fine | All apps' schemas break at once |

**Recipe for the hybrid that most teams converge on:**

- **Mule keeps:** `service.name`, `service.namespace`, `service.version`, `correlation.id`, `client.id`, `http.route`, `user.id`, `trace.id`, `span.id`. Anything that is **per-deployment** or **per-request**.
- **Collector keeps:** `deployment.environment`, `deployment.target`, `deployment.region`, `mule.runtime.version`, `api.layer`, `business.domain`, `data.classification`, `event.outcome`. Anything that is **per-environment** or **derived**.

That split puts each field where the team that *knows the value* sits — and avoids the productivity tax of either extreme.

---

## What We Covered

- We mapped each group from Part 9's taxonomy onto a specific OTel Collector processor — `resource`, `attributes`, `transform`, with `routing` for per-app variation.
- We wrote a `config.yaml` that adds the same eleven-field schema as Part 9 *plus* a derived `event.outcome` field, **entirely on the collector**.
- We covered the `routing` connector for per-app decoration, including the deprecation note on the older `processors.routing`.
- We laid out the **trade-off table and the hybrid recipe** — the practical answer for most teams is *keep per-deployment fields on Mule, push per-environment and derived fields to the collector*.

This post and Part 9 give us two clean implementations of the same schema. The third post in this trio — Part 11 — keeps the implementation but reshapes the *output*: take the OTLP-shaped documents we have been writing all along (`Resource.*`, `Attributes.*`, `Body`, `SeverityText`) and rewrite them into ECS-native JSON (`message`, `log.level`, top-level `service.*`, top-level `host.*`) so Kibana's prebuilt UIs and ILM templates work without runtime fields or aliasing.

> ➡️ **Next up:** In Part 11 we will treat the OTel Collector as a small ETL stage — receive OTLP, **flatten and rename** the Resource and Attributes into ECS-flat field names, drop redundant fields, and write a document Kibana's ECS-native UIs (Logs, APM, Observability) can consume natively. Same schema, much smaller footprint per record, and dashboards that ship with Elastic just work.

---

## References

- [OpenTelemetry Collector — `resource` processor](https://github.com/open-telemetry/opentelemetry-collector/tree/main/processor/resourceprocessor)
- [OpenTelemetry Collector Contrib — `attributes` processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor)
- [OpenTelemetry Collector Contrib — `transform` processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/transformprocessor)
- [OpenTelemetry Collector Contrib — `routing` connector](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/connector/routingconnector)
- [OpenTelemetry Collector Contrib — `filter` processor](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/filterprocessor)
- [OTTL — OpenTelemetry Transformation Language Spec](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/pkg/ottl)
- [OpenTelemetry — Resource Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/resource/)
- [Part 9 — Standardizing the Mule Log Schema — Path A](#)
