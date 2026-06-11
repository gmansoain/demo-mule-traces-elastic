---
title: "Standardizing the Mule Log Schema — Path A — Mule-Side Resource Attributes and MDC — Part 9"
slug: "mule-log-schema-standardization-resource-attributes-mdc"
description: "Define a standard set of fields every Mule log line should carry, classify them by lifetime, and implement the schema entirely on the Mule side with OTel resource attributes for constants and MDC for per-request values."
author: "Gonzalo Marcos"
date: 2026-06-09
status: not validated
lang: en
category: observability
tags:
  - mule-runtime
  - logging
  - best-practices
  - architecture-diagram
  - english
  - series
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 9
type: tutorial
difficulty: intermediate
read_time: 16
mule_version: "4.11"
platform:
  - anypoint-platform
  - cloudhub-2
  - runtime-fabric
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![CloudHub 2.0](https://img.shields.io/badge/Platform-CloudHub_2.0-00A0DF?logo=mulesoft&logoColor=white) ![Runtime Fabric](https://img.shields.io/badge/Platform-Runtime_Fabric-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 9](https://img.shields.io/badge/Part-9-16a085) ![16 min](https://img.shields.io/badge/Read_Time-16_min-lightgrey)

# Standardizing the Mule Log Schema — Path A — Mule-Side Resource Attributes and MDC — Part 9

By the end of [Part 8](#) we had a working dashboard joining `mule-logs` and `mule-traces` on `trace.id`. Logs were *flowing*, but the log records were generic — `Body`, `SeverityText`, `Resource.service.name`, and a handful of OTel-default attributes. A real Mule deployment, even at single-team scale, needs more on every log line: which environment it came from, which API layer, which application version, who the calling client was, what business correlation ID the request belongs to. Without that, free-text search is the only filter we have.

This post is the first of three on **standardizing the log schema** for the stack we built in Parts 1–8. We will look at *why* a shared schema matters even on a small fleet, propose a concrete catalogue of **fifteen-plus fields** we recommend on every Mule log line, classify them by **how often they change** (deployment / transaction / event), and then implement the schema entirely **on the Mule side** — Path A — using OTel **resource attributes** for the constants and **Log4j MDC** for the per-request values. The next two posts swap the implementation point: Part 10 does the same job from the OTel Collector, and Part 11 reshapes OTLP records into ECS-native JSON before they reach Elastic.

If you only read one of the three, read this one. The grouping taxonomy and the field catalogue are referenced verbatim by Parts 10 and 11.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 8 — Unified Kibana Dashboard: Logs and Traces Across Mule Apps](#) · **Next:** Part 10 — Centralizing the Mule Log Schema with the OTel Collector — Path B

> [!WARNING]
> **HTTP-only, demo-grade.** Same posture as the rest of the series. Once we add fields like `clientId` or business identifiers, the privacy bar rises — the headers we are about to ship include enough context to identify a caller. Tighten the security group, rotate the `mule-logger` password, and treat the demo cluster as throwaway accordingly.

---

## What We Will Cover

- Why a standardized log schema is a higher-leverage investment than a better dashboard.
- A concrete catalogue of **15+ fields** every Mule log line should carry, with examples.
- A taxonomy that groups those fields by **lifetime**: deployment-static, request-scoped, event-scoped, and infrastructure-derived.
- How OTel maps each group onto its data model — `Resource.attributes`, `LogRecord.attributes`, MDC, span context.
- **Path A — implement the schema entirely on the Mule side**: static fields via `mule.openTelemetry.exporter.resource.attributes`, per-request fields via Log4j MDC.
- Verify the result in Kibana — every log document carries the schema and the dashboard from Part 8 lights up with new filters.

---

## Why Standardize the Log Schema at All?

Three reasons that all show up in production within the first six months.

### 1. Filtering is queries, not greps

A `mule-logs` document with three fields — `@timestamp`, `Body`, `Resource.service.name` — forces the on-call engineer into free-text search. Ten apps × three environments × two replicas means the same `Body: "Order processed"` line appears in 60 places. With `Resource.environment: "prod"`, `Resource.apiLayer: "experience"`, and `Attributes.client_id: "ios-app-3.4"` on the same record, the same question becomes a four-clause KQL filter that returns the four lines that actually matter.

### 2. Cross-team consistency is a contract, not a coincidence

If the *Orders* team logs `applicationName: "orders-api"` and the *Customers* team logs `app: "customer-service"`, every cross-team Kibana panel needs an OR clause that handles both. Multiply by ten teams and the dashboard becomes unmaintainable. A schema agreed at the platform level — *every* Mule app emits these fields under these exact names — turns the dashboard into a single, dumb query.

### 3. Retention and routing scale with the schema

Once `environment` is on every log, ILM can apply a 7-day retention to `prod` and 30-day to `qa` automatically. Once `apiLayer` is on every log, a Kibana space can show only the panels relevant to the API team. None of that works on free text. Schema is what unlocks the operational layers above the dashboard — and it is much cheaper to standardize *before* there are five years of unstructured documents to reindex.

> [!IMPORTANT]
> **Standardization is platform work, not app work.** The schema lives in *one* document maintained by the observability owner, gets implemented identically in every Mule app's deployment, and changes go through the same review as any other platform change. It is not a per-team decision. If the platform team does not own it, two teams will pick different field names within a quarter and the dashboard will need OR clauses for the rest of the company's life.

---

## A Catalogue of Fields Worth Standardizing on

The list below is an opinionated baseline. Adopt it as-is, or pick the rows that fit your environment — but adopt *something* before the second app ships.

Each row is annotated with a **Shipped by default?** column based on what we actually observe in `mule-logs` and `mule-traces` documents on RTF / CloudHub 2.0:

- **Yes (auto)** — Mule's OTel SDK injects this field on every signal without any configuration. Do **not** redeclare it.
- **Yes, but generic** — Mule writes the field, but with a placeholder value (e.g. `mule-container`). We need to set the property to make it useful.
- **No** — Mule does not write this. We have to add it ourselves (resource attribute, MDC, or DataWeave).

### Identity (who and what is logging)

| Field | Example | Shipped by default? | Why it matters |
| --- | --- | --- | --- |
| `service.name` | `process-payments-sepa` | **Yes, but generic on logs** — Mule writes `mule-container` to `Resource.service.name` on log records regardless of the property; on **traces** it correctly carries the artifact ID. Set `mule.openTelemetry.exporter.resource.service.name` AND duplicate it in `resource.attributes` to override on logs. | The single most-filtered field in any dashboard. OTel-spec; **do not rename**. |
| `service.namespace` | `retail-banking` | **Yes, but defaults to `mule`** — Mule writes `Resource.service.namespace = "mule"` by default. Override via `mule.openTelemetry.exporter.resource.service.namespace`. | Logical grouping above `service.name`; lets you slice by team or product |
| `service.version` | `1.2.0` | **No** — must be set via `resource.attributes` or the `service.version` dedicated property | Pinpoint regressions to a deploy without correlating with git |
| `service.instance.id` | `e741cf53-0af1-4675-...` | **Yes (auto)** — RTF / CH2 SDK fills with a per-replica UUID; **do not redeclare** | Per-pod / per-replica identity |

### Anypoint-platform identifiers (auto-injected on CH2 / RTF)

| Field | Example | Shipped by default? | Why it matters |
| --- | --- | --- | --- |
| `Resource.envId` | `09584395-bb28-48fd-b6f3-a0d5f5034e73` | **Yes (auto)** | Anypoint Environment UUID. Useful for joining against Anypoint API audit logs. |
| `Resource.orgId` | `98b7f48b-...` | **Yes (auto)** | Anypoint Business Group ID — multi-org tenants need this to disambiguate. |
| `Resource.rootOrgId` | `37aa8fe0-...` | **Yes (auto)** | Top-level Anypoint Org ID. Same value across all Business Groups in the org. |
| `Resource.workerId` | `6b5ffdf676-lwtmv` | **Yes (auto)** | The CH2 / RTF replica's pod ID. Sub-deployment level — *which* replica handled this request. |
| `Resource.telemetry.sdk.name` / `language` / `version` | `opentelemetry`, `java`, `1.60.1` | **Yes (auto)** | Identifies the OTel SDK that produced the record. Useful when debugging a runtime upgrade. |

> [!IMPORTANT]
> **These four fields are written by the platform, not by us.** They show up on every log and span document on CloudHub 2.0 and Runtime Fabric whether we ask for them or not — Mule's OTel SDK injects them as part of resource detection. **Do not redeclare them** in `mule.openTelemetry.exporter.resource.attributes` or you will get duplicate keys (and one of the values will be wrong).

### Environment / deployment context

| Field | Example | Shipped by default? | Why it matters |
| --- | --- | --- | --- |
| `deployment.environment` | `prod`, `staging`, `qa` | **No** — must be set via `resource.attributes` | Splits prod from staging without prefixing every index |
| `deployment.target` | `cloudhub-2`, `runtime-fabric`, `standalone` | **No** — must be set | Which Mule runtime topology produced this log |
| `deployment.region` | `eu-central-1` | **No** — must be set (or, on the collector, derived from EC2 metadata) | Pinpoint a regional outage; route alerts |
| `mule.runtime.version` | `4.11.0` | **No** — must be set explicitly. *Note*: `Resource.telemetry.sdk.version` (`1.60.1`) is auto-injected and refers to the **OTel SDK**, not the Mule runtime | Spot regressions tied to a Mule patch |

### API / domain context

| Field | Example | Shipped by default? | Why it matters |
| --- | --- | --- | --- |
| `api.layer` | `experience`, `process`, `system` | **No** — must be set | Lets dashboards filter by API-led layer |
| `api.name` | `customer-onboarding` | **No** | Business-meaningful API identifier (not the artifact ID) |
| `api.version` | `v2` | **No** | Distinguish parallel API versions of the same app |
| `business.domain` | `orders`, `payments`, `shipping` | **No** | The capability the app supports — for cross-team SLO views |

### Per-request / transaction context

| Field | Example | Shipped by default? | Why it matters |
| --- | --- | --- | --- |
| `correlation.id` | `7deb27d0-6415-11f1-...` | **Yes (auto)** — Mule MDC pre-populates this on every flow invocation. Lands on logs as **top-level `correlationId`** (camelCase) and on traces as **top-level `correlation.id`** (dot-separated). | The unifier across logs, traces, and downstream system calls |
| `processorPath` | `main-flow/processors/1` | **Yes (auto)** — Mule MDC pre-populates this. Lands on logs as **top-level `processorPath`**; on traces it appears as **top-level `location`** (different name, same concept). | Where in the Mule flow the line was emitted |
| `trace.id`, `span.id` | hex | **Yes, conditional** — auto-populated as **top-level `TraceId`/`SpanId`** (OTLP-spec) on every record that fires inside a span. Setting `mule.put.trace.id.and.span.id.in.mdc=true` adds parallel **`trace_id`/`span_id`** lowercase MDC duplicates. Lines emitted outside an active span (startup, scheduler, async error handler) carry neither. | OTel-native correlation |
| `thread.id`, `thread.name` | `29`, `[MuleRuntime].uber.04: ...` | **Yes (auto)** — Mule writes top-level `thread.id` / `thread.name` on logs; traces capture both endpoints as `thread.start.id`, `thread.start.name`, `thread.end.name` | Diagnose thread-pool contention |
| `client.id` | `ios-app-3.4`, `partner-acme` | **No** — must be pushed onto MDC from a request header | Who called us — the most useful field for triage you do not have today |
| `http.route` | `/products/{id}` | **No** — must be pushed onto MDC from `attributes.listenerPath` in the flow | The route, not the URL — groups all requests to the same endpoint |
| `http.method`, `http.response.status_code` | `GET`, `200` | **No** — must be pushed onto MDC from `attributes.method` / payload | Standard ECS HTTP fields; Kibana's HTTP UIs key off them |

### Event-scoped / per-line

| Field | Example | Shipped by default? | Why it matters |
| --- | --- | --- | --- |
| `log.level` | `INFO`, `WARN`, `ERROR` | **Yes (auto)** — Mule writes top-level `SeverityText` (string) and `SeverityNumber` (numeric) on every log record. ECS-name `log.level` requires a Part 11 reshape. | Severity classification |
| `log.logger` | `org.mule.runtime.core.internal.processor.LoggerMessageProcessor` | **Yes (auto)** — Mule writes the logger as `Scope.name` on logs (and `mule` on traces). | Which Log4j logger emitted the event |
| `Body` / `message` | `Inbound request received` | **Yes (auto)** — the actual log message lands as `Body`. ECS-name `message` requires a Part 11 reshape. | The thing the developer wrote |
| `event.outcome` | `success`, `failure` | **No** — derive from `SeverityText` in a Part 10 `transform` rule, or set it from the Mule error handler | ECS-friendly flag for KPI panels |

### Span-only (traces index)

| Field | Example | Shipped by default? | Why it matters |
| --- | --- | --- | --- |
| `Name` | `mule:set-payload`, `mule:http:listener` | **Yes (auto)** | The span's identity — what processor or operation produced it |
| `Kind` | `SPAN_KIND_INTERNAL`, `SPAN_KIND_SERVER`, `SPAN_KIND_CLIENT` | **Yes (auto)** | Whether the span is an inbound listener, outbound call, or in-process processor |
| `Duration` | `651` (microseconds) | **Yes (auto)** | Span wall time — the dashboard's hero metric for latency |
| `ParentSpanId` | `e7ae9e8ee0aa0941` | **Yes (auto)** — empty on the root span | Parent-child link that builds the trace tree |
| `TraceStatus` | `0` unset, `1` OK, `2` ERROR | **Yes (auto)** — defaults to `0` unless an error handler marks the span | Span-level status flag |
| `artifact.id`, `artifact.type` | `mule-ds-logs`, `app` | **Yes (auto)** — the real Mule artifact identity that logs are missing | Useful for joining logs and traces back to the same app |

### Compliance / regulatory (only if applicable)

| Field | Example | Shipped by default? | Why it matters |
| --- | --- | --- | --- |
| `data.classification` | `pii`, `phi`, `internal` | **No** | Drives masking rules and ILM retention; required in regulated industries |
| `tenant.id` | `acme-tenants-12` | **No** — must be pushed onto MDC | Multi-tenant SaaS — what one tenant did, isolated |

### Quick reference — what we get for free vs. what we have to add

| Category | Auto-shipped by Mule's OTel SDK | We have to add |
| --- | --- | --- |
| **Identity** | `service.instance.id`, `service.namespace` (defaults to `mule`) | `service.name` (override needed on logs), `service.version` |
| **Anypoint platform** | `envId`, `orgId`, `rootOrgId`, `workerId`, all `telemetry.sdk.*` | nothing |
| **Environment / domain** | nothing | `deployment.environment`, `deployment.target`, `deployment.region`, `mule.runtime.version`, `api.*`, `business.domain` |
| **Per-request — common** | `correlationId` / `correlation.id`, `processorPath` / `location`, `thread.*`, `TraceId` / `SpanId` (when in a span) | `client.id`, `http.route`, `http.method`, `tenant.id`, `user.id` |
| **Per-event** | `Body`, `SeverityText`, `SeverityNumber`, `Scope.name` (logger) | `event.outcome` |
| **Span-only** | `Name`, `Kind`, `Duration`, `ParentSpanId`, `TraceStatus`, `artifact.id`, `artifact.type` | nothing |

That is **30+ fields surveyed**. About **half are shipped by default**; the other half are platform-team work. A reasonable baseline for most Mule deployments lands around ten to twelve fields total — pick from the **"we have to add"** column based on what your dashboards and alerts actually need to filter on.

> [!TIP]
> **Resist adding fields you cannot fill.** Every field you declare is a contract: every team adds it, every code review checks for it, every cluster indexes it. A field that ends up `null` 90% of the time is noise. Start with the eight or ten you know you will use; add more deliberately when a dashboard or runbook actually needs them.

---

## How to Group the Fields — A Lifetime-Based Taxonomy

The catalogue above is long. The reason it is *manageable* is that the fields cluster by **how often they change** — and that classification is what tells us *where* to set them in OTel.

| Group | Lifetime | Examples | OTel home | Mule implementation |
| --- | --- | --- | --- | --- |
| **Deployment-static** | Set once per deployment, stable until next deploy | `service.name`, `service.version`, `deployment.environment`, `api.layer`, `business.domain`, `mule.runtime.version` | `Resource.attributes` | Runtime Manager Properties → `mule.openTelemetry.exporter.resource.attributes` |
| **Infrastructure-derived** | Set automatically by the runtime / platform | `service.instance.id`, `deployment.target`, `host.name`, `process.pid` | `Resource.attributes` (auto) | **Do not redeclare**; OTel SDK fills these |
| **Per-request / transaction** | Set on each inbound request, stable across the flow | `correlation.id`, `client.id`, `user.id`, `http.route`, `http.method`, `tenant.id` | `LogRecord.attributes` (and span attributes) | MDC — `MDC.put("...", value)` once per request |
| **Per-event** | Set on each individual log line | `Body` / `message`, `log.level`, `log.logger`, `event.outcome` | `LogRecord.body` / `LogRecord.severityText` / `LogRecord.attributes` | Log4j event itself |
| **Trace context** | Set per span, propagated by `traceparent` | `trace.id`, `span.id` | `LogRecord.spanContext` | Mule's `mule.put.trace.id.and.span.id.in.mdc=true` from Part 5 |

The taxonomy isn't just a way to organize a wiki page — it dictates **where** the field gets set:

- A deployment-static field set on every event is a waste of bytes (and a maintenance nightmare). Set it once on `Resource` and let OTel ride it on every log and span.
- A per-request field set on `Resource` is plain wrong — `Resource` is supposed to be stable for the lifetime of the SDK. Set it via MDC.
- A per-event field set via MDC is a footgun — MDC is per-thread, and a value pushed in one flow can leak to the next. Per-event values belong on the event itself.

Map every field you adopt to its group **before** you implement, and the wiring becomes mechanical.

---

## Overview — Path A

```
┌────────────────────────────────────────────────────────────────────┐
│  Mule app (Mule 4.11)                                              │
│                                                                    │
│  Runtime Manager Properties                                        │
│    mule.openTelemetry.exporter.resource.service.name=...           │
│    mule.openTelemetry.exporter.resource.attributes=                │
│      deployment.environment=prod,                                  │
│      api.layer=process, ...                                        │
│                       │                                            │
│                       │ becomes Resource.attributes on every       │
│                       │ log record AND every span                  │
│                       ▼                                            │
│  ┌────────────────────────────────────────┐                        │
│  │ Inbound Flow                           │                        │
│  │  ┌────────────┐  ┌────────────────┐    │                        │
│  │  │ HTTP       │─▶│ MDC.put(       │    │                        │
│  │  │ Listener   │  │   client.id,   │    │                        │
│  │  │            │  │   user.id,     │    │                        │
│  │  │            │  │   http.route)  │    │                        │
│  │  └────────────┘  └────────────────┘    │                        │
│  │           │                            │                        │
│  │           ▼                            │                        │
│  │  ┌────────────┐                        │                        │
│  │  │ Logger     │ → LogRecord.attributes │                        │
│  │  │ INFO ...   │   include MDC keys     │                        │
│  │  └────────────┘                        │                        │
│  └────────────────────────────────────────┘                        │
└────────────────────────────────────────────────────────────────────┘
                       │ OTLP/HTTP
                       ▼
            OTel Collector → Elasticsearch
```

The collector and the rest of the Part 5 pipeline are unchanged. The schema lives entirely in Mule's runtime properties and the Log4j MDC.

---

## Table of Contents

1. [Step 1 — Pick the Field Set](#step-1--pick-the-field-set)
2. [Step 2 — Set the Deployment-Static Fields as Resource Attributes](#step-2--set-the-deployment-static-fields-as-resource-attributes)
3. [Step 3 — Set Per-Request Fields via MDC](#step-3--set-per-request-fields-via-mdc)
4. [Step 4 — Update `log4j2.xml` for the On-Disk File Layout](#step-4--update-log4j2xml-for-the-on-disk-file-layout)
5. [Step 5 — Trigger the App and Verify the Schema in Kibana](#step-5--trigger-the-app-and-verify-the-schema-in-kibana)
6. [Step 6 — Refresh the Part 8 Dashboard with the New Filters](#step-6--refresh-the-part-8-dashboard-with-the-new-filters)
7. [Verification](#verification)
8. [Troubleshooting](#troubleshooting)

---

### Step 1 — Pick the Field Set

For this tutorial we will adopt **eleven fields** as the baseline. They cover the questions our Part 8 dashboard already asks, plus the three fields we kept wishing we had during the build (environment, API layer, client identity).

**Deployment-static (set on `Resource`):**

- `service.name` — `process-payments-sepa`
- `service.namespace` — `retail-banking`
- `service.version` — `1.2.0`
- `deployment.environment` — `prod`
- `deployment.target` — `runtime-fabric`
- `api.layer` — `process`
- `business.domain` — `payments`
- `mule.runtime.version` — `4.11.0`

**Per-request (set via MDC):**

- `correlation.id` — Mule sets this automatically; we just expose it
- `client.id` — from the `client_id` header (Mule's default header for the API consumer's identifier on HTTP requests, set by Anypoint API Manager's Client ID Enforcement and similar policies)
- `http.route` — from the route definition

Trace context (`trace.id`, `span.id`) is already on the records from Part 5's `mule.put.trace.id.and.span.id.in.mdc=true`. We do not redeclare it.

The full catalogue and the taxonomy live in the **References** at the bottom of this post and are reused verbatim in Parts 10 and 11.

---

### Step 2 — Set the Deployment-Static Fields as Resource Attributes

These eight fields go into Runtime Manager once per deployment. They land on `Resource.*` of every log record and every span emitted by the app.

**Anypoint Runtime Manager → Applications → `<app>` → Settings → Properties.** Add (or update):

| Property                                                 | Value                                                                                                                                                       |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `mule.openTelemetry.exporter.resource.service.name`      | `process-payments-sepa`                                                                                                                                     |
| `mule.openTelemetry.exporter.resource.service.namespace` | `retail-banking`                                                                                                                                            |
| `mule.openTelemetry.exporter.resource.attributes`        | `service.version=1.2.0,deployment.environment=prod,deployment.target=runtime-fabric,api.layer=process,business.domain=payments,mule.runtime.version=4.11.0` |

Click **Apply Changes**. RTF / CH2 restarts the app once.

> [!IMPORTANT]
> `mule.openTelemetry.exporter.resource.attributes` is **a single property** whose value is a comma-separated list of `key=value` pairs. The MuleSoft docs prescribe this exact format. Use underscores or dashes for values that contain spaces — the parser does not quote-escape, so `Mobile APP` becomes a `Mobile` key with the value `APP` and silently breaks the next pair. If you need values with commas or spaces, use **Path B (Part 10)** — the collector-side `resource` processor handles them correctly via YAML.

> [!TIP]
> `service.name`, `service.namespace`, and `service.version` have **dedicated property names** under `mule.openTelemetry.exporter.resource.service.*`. Use those for the OTel-spec fields rather than stuffing them into `resource.attributes` — same effect, but the dedicated names also feed Kibana's **Observability → APM** UI correctly if you ever swap the OTel Collector for Elastic APM Server.

> [!WARNING]
> **`service.instance.id` is set by the runtime, not by us.** RTF fills it from the pod ID; CH2 fills it from the replica ID. If you also declare `service.instance.id` in `resource.attributes`, you will see two competing values in `mule-logs` documents — the runtime's wins on traces, your value wins on logs, and a query that joins on `service.instance.id` will mismatch. Leave it to the runtime.

---

### Step 3 — Set Per-Request Fields with the Mule Tracing Module

Mule already populates `correlationId` and `processorPath` on the MDC automatically — both ride on every OTel log record at the top level. The new fields are **`client.id`** (lifted from an inbound header) and **`http.route`** (the listener path).

The official, supported way to push values into the MDC is the **Mule Tracing module**. Per the [MuleSoft docs](https://docs.mulesoft.com/mule-runtime/latest/logging-mdc), the module *"enables you to enhance your logs by adding, removing, and clearing variables from the logging context for a given Mule event."* Three operations matter:

| Operation | XML element | Attributes | What it does |
| --- | --- | --- | --- |
| Set Logging Variable | `<tracing:set-logging-variable>` | `variableName`, `value` (DataWeave-capable) | Sets a key in the logging context. Survives across the rest of the event. |
| Remove Logging Variable | `<tracing:remove-logging-variable>` | `variableName` | Removes a single key. |
| Clear Logging Variables | `<tracing:clear-logging-variables>` | none | Removes **all** custom keys. Per the docs: *"This option doesn't remove the processor path or the correlation ID."* — the Mule-managed defaults survive. |


> [!WARNING]
> **CloudHub 1.0 does not support MDC logging.** This entire step (and most of the schema work) only applies to **CloudHub 2.0**, **Runtime Fabric**, and **standalone** runtimes. If you are on CH 1.0, MDC pushes from the Tracing module are silently dropped.

#### 3.1 — Add the Tracing module to the project

In Anypoint Studio: **Mule Palette → Search in Exchange → Mule Tracing module → Add to project**. The Maven plugin adds the dependency to `pom.xml`; the namespace `xmlns:tracing="http://www.mulesoft.org/schema/mule/tracing"` lands automatically in the next XML file you open.

> [!TIP]
> If you are on Anypoint Code Builder or driving the build from Maven directly, the dependency block is the standard MuleSoft pattern — `groupId: com.mulesoft.connectors`, `artifactId: mule-tracing-module`, with `<classifier>mule-plugin</classifier>` and the connector version pinned. The exact coordinates evolve with releases; the Studio "Add to project" path always picks the right version for the runtime you target.

#### 3.2 — Build the `observability-init` subflow

We will use a **subflow** that runs at the start of every public flow. Drop it into a shared XML file so every flow can reference it.

📄 `src/main/mule/observability-init.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mule xmlns="http://www.mulesoft.org/schema/mule/core"
      xmlns:tracing="http://www.mulesoft.org/schema/mule/tracing"
      xmlns:doc="http://www.mulesoft.org/schema/mule/documentation"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
      xsi:schemaLocation="
        http://www.mulesoft.org/schema/mule/core
        http://www.mulesoft.org/schema/mule/core/current/mule.xsd
        http://www.mulesoft.org/schema/mule/tracing
        http://www.mulesoft.org/schema/mule/tracing/current/mule-tracing.xsd">

    <sub-flow name="observability-init">
        <tracing:set-logging-variable
                variableName="client.id"
                value="#[attributes.headers['client_id'] default 'unknown']"/>
        <tracing:set-logging-variable
                variableName="http.route"
                value="#[attributes.listenerPath default attributes.requestPath]"/>
    </sub-flow>

    <sub-flow name="observability-cleanup">
        <tracing:clear-logging-variables/>
    </sub-flow>

</mule>
```

📄 Full file: [./assets/hello-world-mule-direct-stream/src/main/mule/observability-init.xml](./assets/hello-world-mule-direct-stream/src/main/mule/observability-init.xml)

A few things worth knowing:

- `value="#[...]"` accepts any DataWeave expression. The default fallback (`default 'unknown'`) makes the field always-present so KQL filters and runtime fields do not have to handle nulls.
- `<tracing:clear-logging-variables/>` is one element with no attributes — it removes every custom key in one shot. The `correlationId` and `processorPath` Mule-managed entries survive (per the docs), so we do not lose request correlation by clearing.
- For surgical removal of a single key (e.g. masking a PII field after a specific processor) use `<tracing:remove-logging-variable variableName="user.id"/>` instead of clearing everything.

#### 3.3 — Reference the subflows from every public flow

```xml
<flow name="payments-sepa">
    <http:listener config-ref="HTTP_Listener_config" path="/products"/>
    <flow-ref name="observability-init"/>
    <logger level="INFO" message="Inbound request received"/>
    <!-- business logic -->
    <flow-ref name="observability-cleanup"/>
</flow>
```

> [!IMPORTANT]
> **Run cleanup on every exit path, including errors.** The Tracing module scopes the logging context to the Mule event, but the safest pattern is still to wrap the flow body in a `<try>` with the cleanup also called from `<error-handler>`. That way the next event reusing this thread starts with a clean slate even when the happy path is skipped.

---

### Step 4 — Update `log4j2.xml` for the On-Disk File Layout

The on-disk file is for live tail in Runtime Manager / RTF logs view, *not* for Elastic — the OTel exporter taps Log4j's in-process event stream directly. But it is worth making the file's pattern carry the same MDC keys we just set, so the live-tail experience matches the Kibana shape.

The MuleSoft docs recommend `[%MDC]` in the pattern to dump every key in the logging context in one shot — that is what we will use. It expands as `{key1=value1, key2=value2, ...}` and adapts to whatever the Tracing module pushed for that event without us having to enumerate every key explicitly.

📄 `src/main/resources/log4j2.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Configuration>

    <Appenders>
        <RollingFile name="file"
                     fileName="${sys:mule.home}${sys:file.separator}logs${sys:file.separator}process-payments-sepa.log"
                     filePattern="${sys:mule.home}${sys:file.separator}logs${sys:file.separator}process-payments-sepa-%i.log">
            <PatternLayout pattern="%-5p %d{ISO8601} [%t] [%MDC] %c: %m%n"/>
            <SizeBasedTriggeringPolicy size="10 MB"/>
            <DefaultRolloverStrategy max="10"/>
        </RollingFile>
    </Appenders>

    <Loggers>
        <AsyncLogger name="org.mule.service.http" level="WARN"/>
        <AsyncLogger name="org.mule.extension.http" level="WARN"/>
        <AsyncLogger name="org.mule.runtime.core.internal.processor.LoggerMessageProcessor"
                     level="INFO"/>

        <AsyncRoot level="INFO">
            <AppenderRef ref="file"/>
        </AsyncRoot>
    </Loggers>

</Configuration>
```

📄 Full file: [./assets/hello-world-mule-direct-stream/src/main/resources/log4j2.xml](./assets/hello-world-mule-direct-stream/src/main/resources/log4j2.xml)

The `[%MDC]` token expands into a comma-separated map of every key in the logging context. A tailed line looks like:

```text
INFO  2026-06-09T16:14:23.512Z [worker-3] {client.id=ios-app-3.4, http.route=/products, processorPath=payments-sepa/processors/2, correlationId=7deb27d0-..., trace_id=4bf92f3577b3...} org.mule...LoggerMessageProcessor: Inbound request received
```

The `processorPath` and `correlationId` Mule-managed defaults appear automatically, alongside whatever we pushed via the Tracing module. Mental model stays consistent across tail and the OTel-side Kibana view.

> [!TIP]
> Prefer `[%MDC]` over enumerated `%X{key}` lookups. Enumerated lookups silently drop new keys when a future flow adds them; `[%MDC]` always shows everything currently set. The trade-off: if log ingestion downstream parses the file (e.g. a future Filebeat sidecar), the field order is no longer fixed. Pick `%X{key}` over `[%MDC]` only when downstream parsing requires a stable shape.

---

### Step 5 — Trigger the App and Verify the Schema in Kibana

Hit `/products` with the new header:

```bash
curl -s -H "client_id: ios-app-3.4" \
  "https://hello-world-mule-direct-stream-rtf.<rtf-ingress>/products"
```

In Kibana → **Analytics → Discover** → **Mule Logs**. Pick the most recent document. The shape we actually get on CloudHub 2.0 / Runtime Fabric (verified against a real document) — note that **MDC keys land at the top level of `_source`, not under an `Attributes` container**:

```json
{
  "@timestamp": "2026-06-09T16:14:23.512Z",
  "Body": "Inbound request received",
  "Resource": {
    "service": {
      "name": "process-payments-sepa",
      "namespace": "retail-banking",
      "instance": { "id": "e741cf53-0af1-4675-..." }
    },
    "service.version": "1.2.0",
    "deployment.environment": "prod",
    "deployment.target": "runtime-fabric",
    "api.layer": "process",
    "business.domain": "payments",
    "mule.runtime.version": "4.11.0",
    "envId": "09584395-bb28-48fd-b6f3-a0d5f5034e73",
    "orgId": "98b7f48b-6753-4ba8-bcf3-7c0fb76bb6f2",
    "rootOrgId": "37aa8fe0-4188-44c1-a9b5-bdce3ca12f0b",
    "workerId": "6b5ffdf676-lwtmv",
    "telemetry": {
      "sdk": { "name": "opentelemetry", "language": "java", "version": "1.60.1" }
    }
  },
  "Scope": {
    "name": "org.mule.runtime.core.internal.processor.LoggerMessageProcessor",
    "version": ""
  },
  "SeverityText": "INFO",
  "SeverityNumber": 9,
  "TraceFlags": 0,
  "correlationId": "7deb27d0-6415-11f1-b8e5-eec4229eb2b5",
  "processorPath": "payments-sepa/processors/2",
  "client.id": "ios-app-3.4",
  "http.route": "/products",
  "thread": {
    "id": 29,
    "name": "[MuleRuntime].uber.04: [process-payments-sepa].payments-sepa.CPU_LITE @7f145f22"
  },
  "TraceId": "0d468729b2cd0ce90b5f5737dc5b65c2",
  "SpanId": "f46ed99f2f194444",
  "TraceFlags": 1,
  "trace_id": "0d468729b2cd0ce90b5f5737dc5b65c2",
  "span_id": "f46ed99f2f194444",
  "trace_flags": "01"
}
```

Pin the **Resource** field group in Discover so the eight schema-relevant Resource keys are visible by default.

> [!IMPORTANT]
> **Four real-shape findings to internalize before Parts 10–11:**
>
> 1. **MDC keys land at the top level**, not under `Attributes`. `correlationId`, `processorPath`, `client.id`, `http.route`, `thread.id`, `thread.name`, and any custom MDC key we push become *peers* of `Body` and `SeverityText`.
> 2. **Anypoint auto-injects four Resource fields** — `Resource.envId`, `Resource.orgId`, `Resource.rootOrgId`, `Resource.workerId` — on every record. The catalogue at the top of this post lists them; do not redeclare them.
> 3. **Trace context appears twice, in two different field-name shapes.** Once `mule.put.trace.id.and.span.id.in.mdc=true` is set, the OTel SDK writes the OTLP-spec **`TraceId`** / **`SpanId`** / **`TraceFlags`** (capitalized, top-level) on every record, AND the MDC bridge writes the lowercase **`trace_id`** / **`span_id`** / **`trace_flags`** (underscores, not dots) on every record. Same values, two field names, both indexed. We will deduplicate this in Part 11's reshape; for now, just be aware it exists. **The Part 8 dashboard drill-down KQL `Attributes.trace.id : "..."` is wrong against this shape — it should be `TraceId : "..."` (or `trace_id : "..."`)**.
> 4. **If `trace_id` and `TraceId` are *both* missing**, the Logger fired outside an active span (a startup line, a scheduler tick, an async error handler) — `mule.put.trace.id.and.span.id.in.mdc=true` cannot fabricate a trace where there is none. Acceptable; just understand what it means.

> [!NOTE]
> *Screenshot — Kibana Discover with the **Mule Logs** data view, a single document expanded, all eleven schema fields present and the **Resource** and **Attributes** field groups pinned.*

---

### Step 6 — Refresh the Part 8 Dashboard with the New Filters

The Part 8 dashboard already breaks down by `Resource.service.name`. With the new schema we can add three more filters as **dashboard-level controls**:

1. Open the dashboard → **Add filter → Add control**.
2. Add three **Options list** controls, one per field:
   - `Resource.deployment.environment`
   - `Resource.api.layer`
   - `Attributes.client.id`
3. Save the dashboard.

Now any panel filters by environment, by API layer, by calling client — without rebuilding any visualization.

> [!NOTE]
> *Screenshot — Part 8 dashboard with the three new dropdowns at the top, set to `prod` / `experience` / `ios-app-3.4`, all five panels filtered to the matching subset.*

---

## Verification

**1. Every new log document carries the eleven fields.**

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_search" \
  -H 'Content-Type: application/json' \
  -d '{
    "query": { "match_all": {} },
    "size": 1,
    "sort": [{ "@timestamp": "desc" }]
  }' | jq '.hits.hits[0]._source | {
        resource: .Resource,
        body: .Body,
        severity: .SeverityText,
        correlationId,
        processorPath,
        "client.id",
        "http.route",
        thread
      }'
```

We expect the `resource` block to carry every static field from Step 2 plus the four Anypoint auto-injected ones (`envId`, `orgId`, `rootOrgId`, `workerId`), and the top-level fields to carry the per-request values from the MDC.

**2. The MDC bleed test.**

Hit the endpoint twice with **different** `client_id` headers — say `ios-app-3.4` and `android-app-2.0` — within a few seconds. In Kibana, filter by the most recent two documents and confirm `client.id` is the *correct* value for each. If both records carry the same `client.id`, the `<tracing:clear-logging-variables/>` call is missing from the cleanup subflow, or the error path is skipping it.

**3. Resource attributes visible on traces too.**

```bash
curl -u mule-tracer:$MULE_TRACER_PASSWORD \
  "http://10.0.1.20:9200/mule-traces/_search?size=1&sort=@timestamp:desc&pretty" \
  | jq '.hits.hits[0]._source.Resource'
```

`Resource.api.layer`, `Resource.deployment.environment`, and the rest should appear here too — that is the point of putting them on `Resource` instead of on every event. One change applies to logs and traces.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Resource attributes saved in Runtime Manager but not visible on log records | App not restarted after Apply Changes | RTF / CH2 restarts the app on Apply Changes, but if the deployment is in a degraded state the change does not propagate; redeploy or stop/start the app |
| `mule.openTelemetry.exporter.resource.attributes` parses one less field than expected | A value contained a comma or `=` and corrupted parsing | Use Path B (Part 10) for those values, or replace them with a safe substitute on Mule side and rename in the collector |
| `Resource.service.instance.id` shows two competing values | Both runtime auto-fill and a manual entry in `resource.attributes` | Delete the manual entry; the runtime owns this field |
| `client.id` carries the *previous* request's value on some lines | The `<tracing:clear-logging-variables/>` call did not run on this event — typically because an exception path bypassed the cleanup subflow | Wrap the flow body in a `<try>` and call `<flow-ref name="observability-cleanup"/>` from both the success path and the `<error-handler>` so cleanup runs on every path |
| `<tracing:set-logging-variable>` is unrecognized — Studio shows red squiggly | The Mule Tracing module is not on the project's classpath | **Mule Palette → Search in Exchange → Mule Tracing module → Add to project** (Step 3.1); verify the dependency lands in `pom.xml` |
| MDC keys silently dropped on CloudHub 1.0 | MDC logging is **not supported** on CloudHub 1.0 (per the MuleSoft docs) | Migrate the app to CloudHub 2.0 / RTF / standalone, or fall back to per-event JSON in the Logger payload — there is no workaround on CH1 |
| `client.id` is `unknown` for every request | Header name mismatch — Mule's default API-consumer header is `client_id` (lowercase, underscore), set by Anypoint API Manager's Client ID Enforcement policy and inherited by RAML/OAS-generated flows | `attributes.headers['client_id']` works for `Client_id`, `Client_Id`, etc. since DataWeave header lookups on `attributes.headers` are case-insensitive; if the caller sends a different name (`x-client-id`, `X-Api-Client-Id`, ...), update the DataWeave path to match |
| Schema lands on logs but not on traces | Resource attributes always ride on both signals when set via `mule.openTelemetry.exporter.resource.*` | Re-check Step 2; if traces still missing, the tracer exporter is using a different SDK init path — file a support case |
| `service.version` is the old value after a redeploy | The Maven build pinned the old version into the deployment YAML | Either source `service.version` from `${project.version}` in `pom.xml` or update the Runtime Manager property explicitly on each release |
| **`Resource.service.name` is `mule-container` despite the property** | On CH2 / RTF, Mule's resource detector writes `mule-container` first; the user-set property is *supposed* to win, but if the property has a typo (`mule.openTelemetry.exporter.resource.service.name` is the correct casing — camel-case `T`, lowercase `name`) the platform value sticks | Re-check the casing on the Properties tab. If correct, **redeploy the app** — the resource detector reads the property only at SDK init, not on Apply Changes. As a workaround, set the same value via `mule.openTelemetry.exporter.resource.attributes` (`service.name=process-payments-sepa,...`) — that path is read with `insert` semantics that override the detector. |
| **`trace.id` / `span.id` missing from log records** | `mule.put.trace.id.and.span.id.in.mdc=true` not on the app's Properties, or the line was logged on a thread without an active span (startup, scheduler, async error handler) | Add the property; redeploy. For lines that genuinely fire outside a span, accept the missing values — log↔trace correlation only applies to in-flow events |
| **MDC keys land at the top level instead of under `Attributes`** | This is the actual Mule + Direct Telemetry Stream behavior — MDC keys become **top-level** fields on the OTLP record | Update any KQL that previously used `Attributes.trace.id` to the top-level paths — `TraceId`/`SpanId` (OTLP-spec, capitalized) or `trace_id`/`span_id` (MDC bridge, underscores). Update Part 8's drill-down URL accordingly. |
| **Trace context appears as both `TraceId` and `trace_id` on every record** | The OTel SDK writes the OTLP-spec capitalized fields; the MDC bridge **also** writes lowercase underscore-separated copies | Both are correct — pick one as canonical for queries (we recommend `TraceId` since it is the OTLP-spec value) and either ignore the duplicate or remove it via a Part 11 reshape rule |
| **`service.name` on logs is `mule-container` instead of the real app name** | On RTF, the OTel SDK's resource detector for **logs** uses a generic `mule-container` regardless of the `mule.openTelemetry.exporter.resource.service.name` property. **Traces** use the artifact ID correctly. The asymmetry breaks any panel that joins logs to traces on `service.name`. | Two workarounds: (1) put the desired `service.name` into `mule.openTelemetry.exporter.resource.attributes` (`service.name=...,...`) — that path overrides the detector on logs as well; (2) handle it on the collector with a `resource` processor that copies `Resource.artifact.id` (present on traces) or sets `service.name` per `service.namespace` for logs. Approach (2) is in Part 10. |

---

## What We Covered

- Why a standardized log schema is platform-team work and not per-team optimization.
- A 22-field catalogue covering identity, environment, API/domain, transaction, event, and compliance contexts.
- A lifetime-based taxonomy that maps each field to its OTel home: `Resource.attributes` for the static, `LogRecord.attributes` (via MDC) for the per-request, `LogRecord` body / severity for the per-event.
- **Path A** — implement the schema entirely on the Mule side: eight static fields via `mule.openTelemetry.exporter.resource.*` properties, three per-request fields via Log4j MDC populated by an `observability-init` subflow.
- A reusable `observability-init` / `observability-cleanup` subflow pair that any future flow can adopt without re-implementing the wiring.
- The Part 8 dashboard upgraded with three filter controls — environment, API layer, client — without rebuilding any panel.

This is the right approach when the field values are **known per-deployment** and the team that owns the Mule app also owns the schema. The next post — Part 10, *Path B* — implements the same schema **centrally on the OTel Collector**. That is the right approach when one observability team owns the schema across many Mule apps and wants to enforce it without per-app rebuilds.

> ➡️ **Next up:** In Part 10 we will lift the eleven fields out of Mule's properties and put them on the OTel Collector instead, using `resource` and `attributes` processors. Same schema, same Kibana experience — but the schema definition is in one YAML file the platform team owns, not in N Runtime Manager Properties tabs the app teams own. We will compare both paths in a Trade-offs table at the end of Part 10 so it is clear when each fits.

---

## References

### Field catalogue (reused in Parts 10 and 11)

The full 22-field catalogue from this post, grouped by lifetime:

- **Deployment-static (`Resource.attributes`):** `service.name`, `service.namespace`, `service.version`, `deployment.environment`, `deployment.target`, `deployment.region`, `mule.runtime.version`, `api.layer`, `api.name`, `api.version`, `business.domain`, `tenant.id`, `data.classification`.
- **Infrastructure-derived (auto, `Resource.attributes`):** `service.instance.id`, `host.name`, `process.pid`.
- **Per-request (`LogRecord.attributes` via MDC):** `correlation.id`, `client.id`, `user.id`, `http.route`, `http.method`, `http.response.status_code`, `flow.name` / `processorPath`.
- **Per-event (`LogRecord`):** `Body` / `message`, `log.level` / `SeverityText`, `log.logger`, `event.outcome`.
- **Trace context (`LogRecord.spanContext` + MDC bridge):** `trace.id`, `span.id`.

### Documentation

- [OpenTelemetry support for Mule runtime — MuleSoft Docs](https://docs.mulesoft.com/mule-runtime/latest/otel-support)
- [OpenTelemetry — Resource Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/resource/)
- [OpenTelemetry — Log Data Model](https://opentelemetry.io/docs/specs/otel/logs/data-model/)
- [Elastic Common Schema — Fields](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html)
- [SLF4J MDC — Documentation](https://www.slf4j.org/api/org/slf4j/MDC.html)
- [Mule Tracing Module — Anypoint Exchange](https://www.mulesoft.com/exchange)
- [Log4j2 PatternLayout `%X{key}` — Apache Docs](https://logging.apache.org/log4j/2.x/manual/layouts.html#patternlayout)
