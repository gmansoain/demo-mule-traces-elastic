---
title: "Reshaping OTLP Logs into ECS-Native JSON — OTel Collector Processors as ETL — Part 11"
slug: "otel-collector-otlp-to-ecs-elasticsearch-mapping"
description: "Use OTel Collector processors and the elasticsearch exporter's ECS mapping mode to flatten and rename OTLP-shaped documents into ECS-native JSON Kibana's prebuilt UIs and ILM templates can consume natively."
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
  - elasticsearch
  - english
  - series
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 11
type: tutorial
difficulty: advanced
read_time: 17
mule_version: "4.11"
platform:
  - anypoint-platform
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Advanced](https://img.shields.io/badge/Level-Advanced-e74c3c) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 11](https://img.shields.io/badge/Part-11-16a085) ![17 min](https://img.shields.io/badge/Read_Time-17_min-lightgrey)

# Reshaping OTLP Logs into ECS-Native JSON — OTel Collector Processors as ETL — Part 11

[Parts 9](#) and [10](#) gave us **the same schema** in two different homes: Mule-side resource attributes plus MDC, or centralized OTel Collector processors. Either way, the resulting `mule-logs` documents are OTLP-shaped — `Resource.service.name`, `Attributes.trace.id`, `Body`, `SeverityText`. That is fine for the Lens dashboard we built in Part 8, where every field is referenced explicitly. But Elastic ships an ecosystem of UIs and tooling — **Observability → Logs**, **Observability → APM**, ILM templates, prebuilt detection rules, the Logs Explorer's auto-categorization — that all key off **ECS field names**: `message`, `log.level`, top-level `service.name`, `host.hostname`, `process.thread.name`. None of those work out of the box on an OTLP-shaped document.

This post turns the OTel Collector into a small **ETL stage**. We will receive the OTLP record exactly as we have been all along, then flatten and rename it into ECS-native JSON before the `elasticsearch` exporter writes it. Two paths: a **lean recipe** that uses the exporter's built-in `mapping.mode: ecs` for the cases where it does the right thing, and a **full recipe** that uses `transform` + `attributes` processors when we need fine control. Both end with the same outcome: a document Kibana's prebuilt UIs accept without runtime fields, without data-view aliases, and without the `Resource.*` / `Attributes.*` prefix wart.

This is the post where the Collector earns its keep beyond *being a translator*. We start using it as a small streaming database that fixes shapes before they land.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 10 — Standardizing the Mule Log Schema — Path B — Centralizing on the OTel Collector](#) · **Next:** Series wrap-up

> [!WARNING]
> **HTTP-only, demo-grade.** Same posture as the rest of the series. Reshaping fields does not change the wire posture — keep the Mule-to-Collector and Collector-to-Elastic links inside a private VPC.

> [!IMPORTANT]
> **Path A vs Path B from Parts 9–10 is independent of this post.** The reshape recipes here apply equally to documents that came from Mule-side properties (Path A) or from collector decorations (Path B). What matters is that the *input* is OTLP-shaped — which is true of every document we have written since Part 5.

---

## What We Will Cover

- Why OTLP and ECS are *both* "JSON for log events" but **not** drop-in compatible — the field-naming and structural differences.
- The **`elasticsearch` exporter's `mapping.mode`** options (`raw`, `ecs`, `bodymap`) — what each one does to the document, when each one is the right pick.
- A **lean recipe**: switch one line in `config.yaml` and let the exporter do the ECS conversion. Catches 80% of the cases.
- A **full recipe**: use `transform` and `attributes` processors to handle the cases the exporter cannot — custom fields, conditional renames, top-level promotion of nested keys.
- Setting `_index` dynamically per record so logs route by `service.name` or `deployment.environment` without changing the Mule properties.
- Verifying the result in Kibana's **Observability → Logs** UI, which we have not been able to use until now.

---

## Prerequisites

- A working OTel Collector from [Part 5](#), updated through [Parts 9 / 10](#) so the schema is in place.
- The `mule-logs` and `mule-traces` indexes from [Part 3](#) — but we will create **new** ECS-shaped indexes (`mule-logs-ecs`) alongside, so we can compare side by side.
- Kibana ≥ 8.x with the **Logs UI** available (default in `kibana_admin`-permitted spaces).
- The `mule-logger` and `mule-tracer` users from Part 3.
- Collector binary `v0.110.0`+ (the `mapping.mode: ecs` option in the `elasticsearch` exporter has matured significantly through `v0.100`+ — older versions emit subtly different fields).

---

## OTLP vs ECS — Same Information, Different Schema

Both formats describe a log line as JSON. They diverge on three axes.

### 1. Field naming convention

| Concept | OTLP-shaped — actual `_source` from CH2 / RTF | ECS-shaped (what Elastic wants) |
| --- | --- | --- |
| Service identity | `Resource.service.name` | `service.name` (top-level) |
| Service instance | `Resource.service.instance.id` | `service.node.name` |
| Severity (text) | `SeverityText` | `log.level` |
| Severity (number) | `SeverityNumber` | (preserved as `log.syslog.severity.code` if useful) |
| Message | `Body` | `message` |
| Logger / Scope | `Scope.name` | `log.logger` |
| Thread | `thread.name` (top-level — flattened from MDC) | `process.thread.name` |
| Thread ID | `thread.id` (top-level) | `process.thread.id` |
| Trace ID | `TraceId` (top-level on logs) | `trace.id` |
| Span ID | `SpanId` (top-level on logs) | `span.id` |
| Mule correlation | `correlationId` (top-level) | `labels.correlation_id` |
| Mule processor path | `processorPath` (top-level) | `mule.flow.processor_path` |
| Anypoint env / org IDs | `Resource.envId`, `Resource.orgId`, `Resource.rootOrgId` | `labels.anypoint_env_id`, `labels.anypoint_org_id`, `labels.anypoint_root_org_id` |
| Anypoint replica | `Resource.workerId` | `host.id` (or `labels.worker_id`) |

Field naming differs in two systematic ways: ECS prefers **flat dotted-string keys** (`service.name`) at the top level; OTLP groups fields under **JSON-object containers** (`Resource: { "service.name": ... }`). A field named `service.name` *inside* a `Resource` object is, in Elasticsearch's eyes, the path `Resource.service.name` — and **that path is not what ECS-aware UIs query for**.

### 2. Structural shape

OTLP keeps record concerns separated by container — `Resource` for static identity, `Attributes` for per-record key-values, `Scope` for the emitting library, `Body` for the actual message, `SeverityText` and `SeverityNumber` at the top level. ECS flattens everything: `message` and `service.name` and `host.hostname` are all top-level, all under the same root document.

### 3. What Kibana queries against

The Logs UI's "Stream" view does an aggregation on `host.hostname`, color-codes by `log.level`, and shows `message` as the primary column. None of those fields exist on a `mapping.mode: raw` document — the data is there under `Resource.host.name`, `SeverityText`, and `Body`, but the UI does not know to look there.

Same applies to:

- **Observability → APM** — keys off `service.name`, `transaction.name`, `transaction.duration.us` at top level.
- **ILM templates `logs-*`** — Elastic's prebuilt mappings assume ECS field names.
- **Detection rules** — most prebuilt rules query `event.action`, `event.outcome`, `user.name` etc. as top-level fields.

A document written with `mapping.mode: raw` is *queryable* but **not native**. The reshape we do in this post makes it native.

> [!IMPORTANT]
> **ECS is one specific schema, not "JSON".** Picking ECS commits us to its naming conventions for *every* field — including ones not in the spec, where the convention is documented in the [ECS custom fields guide](https://www.elastic.co/guide/en/ecs/current/ecs-custom-fields-in-ecs.html). For our schema from Parts 9–10, this means `business.domain` (already lowercase dot-separated) is fine; `apiLayer` would need to become `api.layer`. Fields that look ambiguous now will look ambiguous in ten years' worth of dashboards if we do not pick the right name first.

---

## The `elasticsearch` Exporter's Three `mapping.mode` Options

The OTel Collector's `elasticsearch` exporter is the bottleneck where shapes get committed to the index. It exposes three modes:

| Mode | What it does | When to use |
| --- | --- | --- |
| **`raw`** | Writes the OTLP record verbatim. `Resource.*`, `Attributes.*`, `Body`, `SeverityText` all preserved. | Default in Parts 5–10. Fine when our dashboards explicitly key off OTLP fields and we do not need ECS UIs. |
| **`ecs`** | Maps the well-known OTLP fields to their ECS equivalents (`Body` → `message`, `SeverityText` → `log.level`, `Resource.service.name` → `service.name`). Custom fields preserved as-is. | When we want native ECS support without writing every rename ourselves. Recommended for most teams. |
| **`bodymap`** | When `Body` is *itself* a JSON object, flattens its contents to top-level fields. | When the Mule app emits structured `Body` payloads (rare with Direct Telemetry Stream; common with Filebeat + JsonTemplateLayout). |

`mapping.mode: ecs` does ~80% of the work for free. The remaining 20% — fields we added in Parts 9–10 like `api.layer` or `business.domain`, which are not in the well-known OTLP→ECS map — needs a `transform` processor to handle.

We will set up both. The lean recipe first; the full recipe immediately after.

---

## Overview — The ETL Pipeline

```
┌──────────────────┐
│  Mule app (4.11) │  OTLP/HTTP, raw shape:
│                  │   Resource.service.name, Body, SeverityText, ...
└────────┬─────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────────┐
│  OTel Collector — ETL pipeline                                  │
│                                                                 │
│   receivers:  otlp                                              │
│                                                                 │
│   processors:                                                   │
│     resource/static       — Parts 9/10 schema                   │
│     transform/promote     — lift custom Resource fields to top  │
│     attributes/rename     — rename leftover OTLP→ECS keys       │
│     transform/index-name  — dynamic _index per record           │
│     batch                                                       │
│                                                                 │
│   exporters:                                                    │
│     elasticsearch/logs-ecs                                      │
│       mapping.mode: ecs   — auto-rename well-known OTLP keys    │
│       logs_dynamic_index: true                                  │
│                                                                 │
└────────────────────────────────────┬────────────────────────────┘
                                     │ HTTP + Basic Auth
                                     ▼
                   ┌──────────────────────────────────┐
                   │  Elasticsearch                   │
                   │   mule-logs-ecs-prod-2026.06     │
                   │   mule-logs-ecs-staging-2026.06  │
                   └──────────────────────────────────┘
```

The Mule app is unchanged from Part 9 / 10. The collector picks up everything else.

---

## Table of Contents

1. [Step 1 — Lean Recipe: Switch the Exporter to `mapping.mode: ecs`](#step-1--lean-recipe-switch-the-exporter-to-mappingmode-ecs)
2. [Step 2 — Inspect What `mapping.mode: ecs` Actually Renames](#step-2--inspect-what-mappingmode-ecs-actually-renames)
3. [Step 3 — Full Recipe: Promote Custom Fields with the `transform` Processor](#step-3--full-recipe-promote-custom-fields-with-the-transform-processor)
4. [Step 4 — Rename Leftover OTLP Keys with the `attributes` Processor](#step-4--rename-leftover-otlp-keys-with-the-attributes-processor)
5. [Step 5 — Dynamic `_index` Routing](#step-5--dynamic-_index-routing)
6. [Step 6 — Verify in Kibana's Logs UI](#step-6--verify-in-kibanas-logs-ui)
7. [Verification](#verification)
8. [Troubleshooting](#troubleshooting)

---

### Step 1 — Lean Recipe: Switch the Exporter to `mapping.mode: ecs`

The fastest possible change: one line. Edit `/etc/otelcol-contrib/config.yaml`:

```yaml
exporters:
  elasticsearch/logs-ecs:
    endpoints: ["http://10.0.1.20:9200"]
    user: mule-logger
    password: ${env:MULE_LOGGER_PASSWORD}
    logs_index: mule-logs-ecs
    mapping:
      mode: ecs           # was: raw
```

Add a parallel pipeline (we keep the original `elasticsearch/logs` for now so we can compare side-by-side):

```yaml
service:
  pipelines:
    logs/ecs:
      receivers: [otlp]
      processors: [resource/static, batch]
      exporters: [elasticsearch/logs-ecs]
```

Validate, restart, hit the app:

```bash
sudo /usr/bin/otelcol-contrib validate --config=/etc/otelcol-contrib/config.yaml
sudo systemctl restart otelcol-contrib
curl -s -H "x-client-id: ios-app-3.4" "https://hello-world-mule-direct-stream-rtf.<rtf-ingress>/products"
```

Now query the new index:

```bash
curl -s -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs-ecs/_search?size=1&sort=@timestamp:desc&pretty" \
  | jq '.hits.hits[0]._source'
```

We expect a document that looks meaningfully different from the `mapping.mode: raw` shape:

```json
{
  "@timestamp": "2026-06-09T18:14:23.512Z",
  "service.name": "process-payments-sepa",
  "service.namespace": "retail-banking",
  "service.version": "1.2.0",
  "service.node.name": "e741cf53-0af1-...",
  "log.level": "INFO",
  "log.logger": "org.mule.runtime.core.internal.processor.LoggerMessageProcessor",
  "message": "Inbound request received",
  "deployment.environment": "prod",
  "deployment.target": "runtime-fabric",
  "api.layer": "process",
  "business.domain": "payments",
  "client.id": "ios-app-3.4",
  "http.route": "/products",
  "thread": { "id": 29, "name": "[MuleRuntime].uber.04: ..." },
  "correlationId": "7deb27d0-...",
  "processorPath": "payments-sepa/processors/2",
  "Resource": {
    "envId": "09584395-...",
    "orgId": "98b7f48b-...",
    "rootOrgId": "37aa8fe0-...",
    "workerId": "6b5ffdf676-lwtmv"
  }
}
```

Four things to notice:

- **The well-known OTLP fields became flat-named ECS**: `Body` → `message`, `SeverityText` → `log.level`, `Scope.name` → `log.logger`, `Resource.service.name` → `service.name`, `Resource.service.instance.id` → `service.node.name`. The exporter did all of that automatically.
- **Our custom fields from Parts 9–10 are still there** (`deployment.environment`, `api.layer`, `business.domain`, `client.id`) — they were already lowercase-dot-separated, so they did not need renaming.
- **Several leftover OTLP-flavored fields stay verbatim**: `correlationId` (camelCase MDC default), `processorPath`, `thread.*` (still nested as an object instead of `process.thread.*`), and the four Anypoint platform fields under `Resource.*`. These are not in the well-known map; we will handle them in Steps 3–4.
- **`trace.id` and `span.id` may or may not be there.** The lean recipe maps them when they exist on the OTLP record. If `mule.put.trace.id.and.span.id.in.mdc=true` is missing on the app or the line was logged outside an active span, they will not appear — same as in Part 9.

> [!IMPORTANT]
> The exporter's `mapping.mode: ecs` is **conservative**: it only renames the OTLP fields that are part of the OTel↔ECS well-known mapping table. It does **not** invent renames for our custom fields, and it does **not** drop any data. Anything it does not know how to rename gets written verbatim. That is why the lean recipe is enough for most cases — and why `transform` is needed for the rest.

---

### Step 2 — Inspect What `mapping.mode: ecs` Actually Renames

Worth knowing exactly which OTLP fields the exporter touches, so we know what is left for us to clean up. The current well-known mapping (as of `v0.110`):

| OTLP field | ECS field |
| --- | --- |
| `Body` | `message` |
| `SeverityText` | `log.level` |
| `SeverityNumber` | (preserved as-is, useful for ordering) |
| `Scope.name` | `log.logger` |
| `TraceId` | `trace.id` |
| `SpanId` | `span.id` |
| `Resource.service.*` | top-level `service.*` (with `service.instance.id` → `service.node.name`) |
| `Resource.host.*` | top-level `host.*` (with `host.name` → `host.hostname`) |
| `Resource.process.*` | top-level `process.*` |
| `Resource.os.*` | top-level `os.*` |
| `Resource.cloud.*` | top-level `cloud.*` |
| `Resource.k8s.*` | top-level `kubernetes.*` |
| `Attributes.exception.*` | `error.*` |

The full list lives in the exporter's `model.go` source — see the **References** section. Read it once, take a picture of it, refer back when in doubt.

What the table does **not** include — and so will need our help in Steps 3–4:

- **Mule MDC defaults that land at top level:** `correlationId`, `processorPath`, `thread.id`, `thread.name`, plus any custom MDC key we push via the `observability-init` subflow from Part 9. The exporter does not know to map `thread.*` → `process.thread.*` because Mule writes them as flat MDC keys, not as the OTel-spec `Attributes.thread.*` block the exporter expects.
- **Anypoint platform Resource attributes:** `Resource.envId`, `Resource.orgId`, `Resource.rootOrgId`, `Resource.workerId`. These are Mule-specific; not in the OTel-spec semantic conventions, so the exporter leaves them under `Resource.*`.
- **Custom Resource attributes from Parts 9–10:** `api.layer`, `business.domain`, `deployment.region`, etc. Already lowercase-dot-separated, so they pass through verbatim. No work needed.
- **Anything else we add via Mule MDC** that is not in the well-known table.

---

### Step 3 — Full Recipe: Promote Custom Fields with the `transform` Processor

The `mapping.mode: ecs` only touches the well-known set. For our custom fields that *are* in the right shape, no work is needed. For ones that need renaming or restructuring, OTTL is the tool.

Five renames worth doing for the document we actually get from CH2 / RTF:

- `correlationId` (top-level, Mule MDC default) → `labels.correlation_id` (ECS pattern for opaque IDs)
- `processorPath` (top-level) → `mule.flow.processor_path` (custom namespace, ECS-style)
- `thread.id` / `thread.name` (top-level Mule MDC) → `process.thread.id` / `process.thread.name` (ECS-spec)
- `Resource.envId` / `Resource.orgId` / `Resource.rootOrgId` → `labels.anypoint_env_id` / `labels.anypoint_org_id` / `labels.anypoint_root_org_id`
- `Resource.workerId` → `host.id` (or `labels.worker_id` if `host.id` is already used)

Recall from Part 10's "OTLP layer vs `_source` layer" sidebar — **OTTL paths use the OTLP layer**, where MDC-derived fields live as `attributes[...]` (even though the exporter flattens them to top-level in `_source`). Resource-level fields live as `resource.attributes[...]`.

Edit the collector config:

```yaml
processors:
  # ... earlier processors ...

  transform/promote-custom:
    error_mode: ignore
    log_statements:
      - context: log
        statements:
          # Move correlationId → labels.correlation_id
          - set(attributes["labels.correlation_id"], attributes["correlationId"]) where attributes["correlationId"] != nil
          - delete_key(attributes, "correlationId")

          # Rename processorPath → mule.flow.processor_path
          - set(attributes["mule.flow.processor_path"], attributes["processorPath"]) where attributes["processorPath"] != nil
          - delete_key(attributes, "processorPath")

          # Restructure thread.* (top-level on Mule) → process.thread.* (ECS)
          - set(attributes["process.thread.id"], attributes["thread.id"]) where attributes["thread.id"] != nil
          - set(attributes["process.thread.name"], attributes["thread.name"]) where attributes["thread.name"] != nil
          - delete_key(attributes, "thread.id")
          - delete_key(attributes, "thread.name")

  transform/promote-anypoint-resource:
    error_mode: ignore
    log_statements:
      - context: resource
        statements:
          # Anypoint platform IDs → ECS labels
          - set(attributes["labels.anypoint_env_id"], attributes["envId"]) where attributes["envId"] != nil
          - set(attributes["labels.anypoint_org_id"], attributes["orgId"]) where attributes["orgId"] != nil
          - set(attributes["labels.anypoint_root_org_id"], attributes["rootOrgId"]) where attributes["rootOrgId"] != nil
          - delete_key(attributes, "envId")
          - delete_key(attributes, "orgId")
          - delete_key(attributes, "rootOrgId")

          # Worker / replica ID → host.id
          - set(attributes["host.id"], attributes["workerId"]) where attributes["workerId"] != nil
          - delete_key(attributes, "workerId")
```

Two separate processors because `transform/promote-custom` operates on `LogRecord.attributes` (the per-event MDC keys) and `transform/promote-anypoint-resource` operates on `Resource.attributes` (which is the second processor's `context: resource`). Wrong context = silent no-op.

Add both to the pipeline:

```yaml
service:
  pipelines:
    logs/ecs:
      receivers: [otlp]
      processors:
        - resource/static
        - transform/promote-anypoint-resource
        - transform/promote-custom
        - batch
      exporters: [elasticsearch/logs-ecs]
```

After restart, the same document now has the ECS-shaped fields and is missing the originals:

```json
{
  "@timestamp": "...",
  "message": "Inbound request received",
  "log.level": "INFO",
  "log.logger": "org.mule.runtime.core.internal.processor.LoggerMessageProcessor",
  "service.name": "process-payments-sepa",
  "service.namespace": "retail-banking",
  "service.version": "1.2.0",
  "service.node.name": "e741cf53-0af1-...",
  "host.id": "6b5ffdf676-lwtmv",
  "process.thread.id": 29,
  "process.thread.name": "[MuleRuntime].uber.04: ...",
  "labels.correlation_id": "cf065f10-...",
  "labels.anypoint_env_id": "09584395-...",
  "labels.anypoint_org_id": "98b7f48b-...",
  "labels.anypoint_root_org_id": "37aa8fe0-...",
  "mule.flow.processor_path": "main-flow/processors/1",
  "deployment.environment": "prod",
  "deployment.target": "runtime-fabric",
  "api.layer": "process",
  "business.domain": "payments",
  "client.id": "ios-app-3.4",
  "http.route": "/products"
}
```

Notice: no `correlationId`, no `processorPath`, no `thread.*` (the original top-level ones), no `Resource.envId/orgId/rootOrgId/workerId`. Every field from the original record either has an ECS home or got dropped deliberately.

> [!TIP]
> `labels.*` is the ECS pattern for **opaque user-defined string fields**. It is meant for things you want to filter on but do not want to commit to a top-level ECS schema. Anything under `labels.foo` is indexed as `keyword` and never analyzed — perfect for IDs and short codes. For numeric per-record metrics use `numeric_labels.*`.

> [!IMPORTANT]
> **Run the `transform` processors before the `mapping.mode: ecs` exporter.** OTTL operates at the OTLP layer; the exporter then writes the renamed attributes (whose keys now follow ECS naming) at top level of `_source`. If we put the transforms after the exporter — there is no "after" — they would not run at all.

#### Trace-side reshape: a separate transform with different field names

Mule's logs and traces use **different field names** for the same concepts (see the table at the top of Part 10). The OTTL above is `context: log` and only fires on log records. We need a parallel `context: span` block to handle the trace shape:

```yaml
processors:
  # ... earlier processors ...

  transform/promote-custom-spans:
    error_mode: ignore
    trace_statements:
      - context: span
        statements:
          # correlation.id (dot-separated on traces) → labels.correlation_id (consistent with logs)
          - set(attributes["labels.correlation_id"], attributes["correlation.id"]) where attributes["correlation.id"] != nil
          - delete_key(attributes, "correlation.id")

          # location (traces) → mule.flow.processor_path (consistent with logs after promote-custom)
          - set(attributes["mule.flow.processor_path"], attributes["location"]) where attributes["location"] != nil
          - delete_key(attributes, "location")

          # Mule artifact identity → ECS-style service.* attributes
          # (the artifact.id is also the *real* service.name, replacing "mule-container" on logs)
          - set(attributes["service.name"], attributes["artifact.id"]) where attributes["artifact.id"] != nil
          - set(attributes["mule.artifact.type"], attributes["artifact.type"]) where attributes["artifact.type"] != nil
          - delete_key(attributes, "artifact.id")
          - delete_key(attributes, "artifact.type")

          # thread.start.* / thread.end.* → process.thread.* (ECS) — pick start as the canonical
          - set(attributes["process.thread.id"], attributes["thread.start.id"]) where attributes["thread.start.id"] != nil
          - set(attributes["process.thread.name"], attributes["thread.start.name"]) where attributes["thread.start.name"] != nil
          - set(attributes["process.thread.end_name"], attributes["thread.end.name"]) where attributes["thread.end.name"] != nil
          - delete_key(attributes, "thread.start.id")
          - delete_key(attributes, "thread.start.name")
          - delete_key(attributes, "thread.end.name")
```

Add it to the **traces** pipeline (mirror the logs pipeline structure from Part 10):

```yaml
service:
  pipelines:
    traces/ecs:
      receivers: [otlp]
      processors:
        - resource/static
        - transform/promote-anypoint-resource
        - transform/promote-custom-spans
        - batch
      exporters: [elasticsearch/traces-ecs]
```

> [!IMPORTANT]
> **`promote-custom` is `log_statements` with `context: log`. `promote-custom-spans` is `trace_statements` with `context: span`.** The two are not interchangeable — applying a span-context OTTL rule to a log pipeline is a silent no-op. Always pair the keyword (`log_statements` / `trace_statements` / `metric_statements`) with the matching context.

> [!TIP]
> **The `set(attributes["service.name"], attributes["artifact.id"])` line is the fix for the `mule-container` problem.** Mule's logs put a generic `mule-container` in `Resource.service.name`; traces put the real artifact ID in `artifact.id`. Promoting `artifact.id` to `service.name` on the trace side at least makes traces canonical. To fix logs, do the same on the log side via `mule.openTelemetry.exporter.resource.attributes` (Path A, see Part 9 troubleshooting) — there is no `artifact.id` attribute on the log record to copy from.

---

### Step 4 — Rename Leftover OTLP Keys with the `attributes` Processor

Some renames are simple enough that `transform`'s OTTL is overkill — a flat key-to-key rename. Use the lighter `attributes` processor for those:

```yaml
processors:
  # ...

  attributes/cleanup:
    actions:
      - key: client_id           # if Mule MDC pushed snake_case
        action: insert
        from_attribute: client.id
      - key: client.id
        action: delete

      - key: event.kind
        value: log
        action: insert
```

`from_attribute` reads from another attribute's value; `value` writes a literal. The processor evaluates actions in order, so `insert` of the new key happens before the `delete` of the old one — same record ends up correctly normalized.

```yaml
service:
  pipelines:
    logs/ecs:
      receivers: [otlp]
      processors:
        - resource/static
        - transform/promote-custom
        - attributes/cleanup
        - batch
      exporters: [elasticsearch/logs-ecs]
```

> [!TIP]
> When a rename is **conditional** (only do it when X is true), `transform` is the right tool. When it is **unconditional**, `attributes` is faster — both for the operator (less to read) and for the runtime (`attributes` is a simpler instruction set than OTTL).

---

### Step 5 — Dynamic `_index` Routing

The most powerful trick the `elasticsearch` exporter offers — write each record to a **different index** based on a field on the record. Two reasons to do this:

- **Per-environment retention** — `mule-logs-ecs-prod-*` keeps 7 days; `mule-logs-ecs-staging-*` keeps 30. ILM applies different policies to different patterns.
- **Per-team isolation** — `mule-logs-ecs-payments-*` is readable only by the Payments team's role; `mule-logs-ecs-accounts-*` only by the Core-banking team.

Enable dynamic indexing:

```yaml
exporters:
  elasticsearch/logs-ecs:
    endpoints: ["http://10.0.1.20:9200"]
    user: mule-logger
    password: ${env:MULE_LOGGER_PASSWORD}
    logs_index: mule-logs-ecs                   # default if no override
    logs_dynamic_index:
      enabled: true
    mapping:
      mode: ecs
```

Then set the per-record index name with the `transform` processor:

```yaml
processors:
  # ...

  transform/index-name:
    error_mode: ignore
    log_statements:
      - context: log
        statements:
          - set(attributes["data_stream.dataset"], "mule")
          - set(attributes["data_stream.namespace"], resource.attributes["deployment.environment"])
```

The exporter looks for `data_stream.dataset` and `data_stream.namespace` (or the `elasticsearch.index` attribute as an alternative) on every log record and routes accordingly. With the snippet above, a `prod` record lands in `logs-mule-prod` (the data stream pattern), a `staging` record lands in `logs-mule-staging`, and so on — all from one config, no per-pipeline duplication.

> [!IMPORTANT]
> Dynamic index routing combined with the new index pattern (`logs-mule-<env>`) means we should **plan our role and ILM model around the new pattern**, not retrofit. Update the `mule-logger` role from Part 3 to allow `write` on `logs-mule-*` and `logs-traces-*` instead of (or in addition to) the original `mule-logs` / `mule-traces`. The OTel exporter creates the indexes on first write thanks to `auto_configure`.

> [!TIP]
> The pattern `logs-<dataset>-<namespace>` is Elastic's standard data-stream naming convention. Using `mule` as the dataset and `prod` / `staging` as the namespace is exactly what the **Logs Explorer** UI parses to populate its environment filter. Stick to the convention — UIs reward you for it.

---

### Step 6 — Verify in Kibana's Logs UI

Until now we have only used Discover. With ECS-shaped documents, **Observability → Logs** finally works.

1. Kibana → **Observability → Logs → Stream**.
2. Click into the data view dropdown — pick the new **`logs-mule-*`** pattern.
3. Set the time filter to **Last 15 minutes** and trigger a couple of `/products` calls.

What we expect:

- Each row shows the **`message`** field as the primary column — not `Body`.
- A color-coded **severity** indicator next to each row, computed from `log.level`.
- A **filter panel on the left** with `host.hostname`, `service.name`, `log.level`, `service.environment`, **all populated** because the field names are now ECS-native.
- Click a row → the detail pane includes a **"View in APM"** link if the document carries `trace.id` (it does — Part 5's MDC bridge gave us that).

> [!NOTE]
> *Screenshot — Kibana's Logs Stream UI showing rows from `logs-mule-*` with the message as the main column, environment filter populated, and a "View in APM" link in the detail panel.*

The same documents are *also* available in Discover with the `mule-logs-ecs-*` data view — the Logs UI just gives a different lens.

---

## Verification

**1. Documents in `logs-mule-*` are ECS-flat.**

```bash
curl -s -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/logs-mule-*/_search?size=1&sort=@timestamp:desc&pretty" \
  | jq '.hits.hits[0]._source | keys | sort'
```

We expect to see top-level keys: `@timestamp`, `host.hostname`, `log.level`, `log.logger`, `message`, `process.thread.id`, `process.thread.name`, `service.name`, `service.namespace`, `service.version`, `trace.id`, `span.id`, plus the customs (`api.layer`, `business.domain`, `deployment.environment`, `client.id`, etc.). No `Resource.*`, no `Attributes.*`, no `Body`, no `SeverityText`.

**2. Routing by environment works.**

```bash
curl -s -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/_cat/indices/logs-mule-*?v"
```

We expect at least `logs-mule-prod` and any other environment we have apps in. Each only contains records of its own environment.

**3. Logs UI's left filter panel is populated, not empty.**

Visual check in Kibana → Observability → Logs → Stream. The filter panel shows clickable counts per `service.name`, per `log.level`, per environment. On a `mapping.mode: raw` document, this panel is empty because the Logs UI does not know how to look at `Resource.service.name`.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| New `logs-mule-prod` index does not appear | `logs_dynamic_index.enabled: false` (default) or `data_stream.namespace` resolves to `nil` | Confirm the dynamic-index block in the exporter; print one OTLP record's resource attributes via `debug` exporter to confirm `deployment.environment` is populated |
| Field still appears under `Resource.*` in the new index | A custom field that the exporter does not auto-promote | Add a `transform` rule like Step 3 to lift it to top level |
| `message` field is `null`; `Body` still present | Mule pushed a structured body (a JSON object), and `mapping.mode: ecs` does not flatten it. | Switch to `mapping.mode: bodymap` for those records, or set the Mule logger to emit a string body (Logger component's "Message" field as a plain string) |
| `process.thread.name` field is missing after `mapping.mode: ecs` alone | Mule writes `thread.id` and `thread.name` as **flat top-level MDC keys**, not as the OTel-spec `Attributes.thread.*` block the exporter looks for | Add the `transform/promote-custom` rule from Step 3 — it copies `attributes["thread.id"]`/`thread.name` (top-level on the OTLP record) into the ECS-spec `process.thread.*` paths |
| Logs UI shows "no data" but Discover sees rows | Logs UI relies on ECS-aware data views; the data view's index pattern needs to match | Create an explicit data view named `logs-mule-*` with time field `@timestamp` |
| "View in APM" link 404s | APM data streams not configured because we did not deploy APM Server | Out of scope — install Elastic APM Server (next series) to wire this up |
| Lots of `mapper_parsing_exception`s after enabling ECS mode | An ECS field collides with an existing dynamic mapping in the index | Always **write to a new index** (`mule-logs-ecs`, not the old `mule-logs`) when changing mapping mode; never reuse an index that was first written by a different mode |
| Custom fields work in Discover but not in Lens autocomplete | Lens caches field metadata; new fields take a refresh | **Stack Management → Data Views → \<data view\> → Refresh field list** |

---

## What We Covered

- The OTLP-vs-ECS naming and structural differences, with a side-by-side field map.
- The three `mapping.mode` options of the `elasticsearch` exporter — `raw`, `ecs`, `bodymap` — and the well-known field map `mode: ecs` applies automatically.
- A **lean recipe** that flips one line in the exporter config and gets 80% of ECS conversion for free.
- A **full recipe** that combines `transform` and `attributes` processors to handle the cases the exporter cannot — custom field promotion, conditional renames, top-level lifting.
- Dynamic `_index` routing so logs land in `logs-mule-<env>` data streams, ready for ILM and per-team role scoping.
- Verification through Kibana's **Observability → Logs** UI, which only became usable once the documents were ECS-shaped.

We now have logs that **Elastic's prebuilt UIs and templates accept natively** — no runtime fields, no data-view aliasing, no field-name guessing. The dashboard from Part 8 can run unchanged on the new index by pointing its data view at `logs-mule-*`; the same filter panels work; and the Logs Stream UI now is a viable on-call surface alongside the dashboard.

This closes the trio on log standardization. Combined with Parts 9 and 10, we have:

- **A schema** worth standardizing on.
- **Two implementation paths** for that schema (Mule-side, collector-side, or hybrid).
- **A reshape pipeline** that turns the implementation into ECS-native documents Kibana's ecosystem natively understands.

> ➡️ **Coming next — a follow-up series.** *"From OTel Collector to Elastic APM"* swaps the OTel Collector + custom indexes for **Elastic APM Server**, which natively ingests OTLP, writes to APM data streams, and unlocks Kibana's full Observability suite — service maps, latency distributions, error groupings, span flame graphs. APM Server understands ECS *and* OTLP semantic conventions natively, so the reshape work in this post becomes optional. Same Mule properties; one fewer component to operate; deeper Kibana-native experience.

---

## References

- [OpenTelemetry Collector Contrib — `elasticsearch` exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/elasticsearchexporter)
- [Elasticsearch exporter — `mapping.mode` source (`model.go`)](https://github.com/open-telemetry/opentelemetry-collector-contrib/blob/main/exporter/elasticsearchexporter/model.go)
- [Elastic Common Schema — Field Reference](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html)
- [ECS — Custom Fields Guide](https://www.elastic.co/guide/en/ecs/current/ecs-custom-fields-in-ecs.html)
- [OTel ↔ ECS — Resource & Attributes Mapping (community wiki)](https://github.com/elastic/ecs/blob/main/docs/otel-alignment.md)
- [OTTL — OpenTelemetry Transformation Language Spec](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/pkg/ottl)
- [Elasticsearch Data Streams — Naming Convention](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html#data-streams-naming-scheme)
- [Kibana Logs UI — Documentation](https://www.elastic.co/guide/en/observability/current/monitor-logs.html)
- [Part 9 — Standardizing the Mule Log Schema — Path A](#)
- [Part 10 — Standardizing the Mule Log Schema — Path B — Centralizing on the OTel Collector](#)
