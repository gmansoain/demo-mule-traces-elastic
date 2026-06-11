---
title: "Unified Kibana Dashboard — Logs and Traces Across Mule Apps — Part 8"
slug: "kibana-dashboard-mule-logs-traces-unified"
description: "Build a single Kibana dashboard that joins mule-logs and mule-traces on trace.id — log volume by service, error rate, slow-trace table with click-through to the matching log lines — and export it as NDJSON."
author: "Gonzalo Marcos"
date: 2026-06-09
status: not validated
lang: en
category: observability
tags:
  - kibana
  - logging
  - architecture-diagram
  - kpis
  - best-practices
  - english
  - series
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 8
type: tutorial
difficulty: intermediate
read_time: 14
mule_version: "4.11"
platform:
  - anypoint-platform
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 8](https://img.shields.io/badge/Part-8-16a085) ![14 min](https://img.shields.io/badge/Read_Time-14_min-lightgrey)

# Unified Kibana Dashboard — Logs and Traces Across Mule Apps — Part 8

Across [Parts 1–7](#) we built the whole stack: Elasticsearch and Kibana on AWS, two least-privilege service users, a Mule 4.11 app shipping logs through Log4j2 (Part 4) and another shipping them as OTLP via Direct Telemetry Stream (Part 5), then a second app that turned a single request into a multi-span distributed trace (Part 7). Logs live in `mule-logs`. Traces live in `mule-traces`. Every record from a given request shares the same `trace.id`. The data is *correlatable* — but until we put it on a single screen, the correlation is theoretical.

In this tutorial we will assemble a **unified Kibana dashboard** with five panels that answer the questions an on-call engineer actually has: *which services are producing logs right now? Which traces are slow? When something is broken, where in the trace tree did it happen, and what did each span log along the way?* We will use **Lens** for visualizations and a **Discover session** with a saved drill-down for the click-through from a slow trace to its log lines. Then we will export the dashboard as NDJSON and ship it as a repo asset so any future cluster can import it in one click.

This is the post where the series payoff finally lands on a screen.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 7 — Distributed Traces from Mule via OpenTelemetry → OTel Collector → Elasticsearch](#) · **Next:** Series wrap-up + future series teaser (Elastic APM Server)

> [!WARNING]
> **HTTP-only, demo-grade.** This dashboard runs against the same plain-HTTP cluster from Part 1. The dashboard NDJSON is portable to any Elastic cluster, but if you are importing it into a TLS-secured one, the data views still need pointing at the right index patterns there. Nothing in this post needs to change for the production variant — the dashboard is purely a query layer.

---

## What We Will Cover

- Confirm both data views (`Mule Logs`, `Mule Traces`) are present and that `@timestamp` drives the time filter on both.
- Build five panels in **Lens**:
  1. **Log volume by `service.name`** — stacked area, which app is talking when.
  2. **Error rate per service** — count of `Severity: ERROR` divided by total, per service, per minute.
  3. **Span duration histogram by service** — span `Duration` distribution, broken out by `Resource.service.name`.
  4. **Slow traces table** — the top 20 traces by total duration, with `TraceId`, `service.name`, and root-span name.
  5. **Trace volume vs error spans** — a two-line time series for total spans and `StatusCode = ERROR` spans.
- Wire a **drill-down** from the Slow Traces table to a Discover session that filters `mule-logs` by the clicked `TraceId` — the click-through from "this trace was slow" to "here are its log lines, in order".
- Export the dashboard, the data views, and the saved Discover session as NDJSON.
- Commit the NDJSON to the series repo so future readers (and our own future clusters) get a one-click import.

---

## Prerequisites

Before we start, we will need:

- A working Elasticsearch + Kibana from [Parts 1–2](#).
- The `mule-logs` and `mule-traces` indexes from [Part 3](#), each with at least a few hundred recent documents — easiest to generate by running the Part 7 setup for ten minutes with a `for i in {1..200}; do curl ...; done` loop in the background.
- Both data views from Part 3 / Part 7, with `@timestamp` set as the **time field**.
- A Kibana user with at least the `kibana_admin` role (we will use `elastic` for the demo). Saving dashboards needs `kibana_admin`; the `mule-logger` and `mule-tracer` service users are write-only and cannot save Kibana objects.

We will sanity-check both data views point at non-empty indexes:

```bash
curl -u elastic:$ELASTIC_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_count?pretty"
curl -u elastic:$ELASTIC_PASSWORD \
  "http://10.0.1.20:9200/mule-traces/_count?pretty"
```

We expect non-zero counts on both. If either is zero, drive traffic against App A from Part 7 first.

---

## Field Names — One Source of Truth Before We Build

The dashboard's queries reference field names in both indexes. Different OTel Collector versions emit slightly different shapes (`Resource.service.name` vs `resource.attributes.service.name`, `Body` vs `body`), and Part 4's Log4j layout uses `message` and `level`. We will pick **one set of field names** and let the dashboard target those — anything that does not match gets a runtime field on the data view to alias it across.

Open Kibana → **Stack Management → Data Views → Mule Logs**. Click into a sample document and confirm the field names. The table below is the shape we expect from Part 7's OTel Collector with `mapping.mode: raw` (the default we used). If your fields differ, **add a runtime field** rather than changing the dashboard.

| Concept | Field on `mule-logs` | Field on `mule-traces` |
| --- | --- | --- |
| Service identity | `Resource.service.name` (writes as `mule-container` on RTF — see callout) | `Resource.service.name` (real artifact ID, e.g. `mule-ds-logs`) |
| Trace ID | `TraceId` (top-level, OTLP-spec) — also duplicated as `trace_id` if MDC bridge is on | `TraceId` (top-level) |
| Span ID | `SpanId` (top-level) — also `span_id` if MDC bridge is on | `SpanId` |
| Severity / status | `SeverityText` (`INFO`, `ERROR`, …) | `TraceStatus` (numeric: `0` unset, `1` OK, `2` ERROR) |
| Message / span name | `Body` | `Name` |
| Span kind | n/a | `Kind` (`SPAN_KIND_INTERNAL`, `SPAN_KIND_SERVER`, `SPAN_KIND_CLIENT`) |
| Duration | n/a | `Duration` (**microseconds** — we will convert to ms with a runtime field) |
| Span timestamps | `@timestamp` only | `@timestamp` (start) + `EndTimestamp` (end) |
| Mule artifact | `Resource.service.name` (generic on logs) | `artifact.id` + `artifact.type` (top-level, real artifact identity) |
| Mule flow location | `processorPath` (top-level) | `location` (top-level) — same concept, different field name |
| Correlation ID | `correlationId` (top-level, camelCase) | `correlation.id` (top-level, dot-separated) |

> [!IMPORTANT]
> **Trace ID is `TraceId` on BOTH indexes** — the assumption in earlier drafts that logs used `Attributes.trace.id` was wrong. The OTel Collector's Elasticsearch exporter under `mapping.mode: raw` flattens the OTLP top-level `TraceId` field straight to `_source.TraceId` on logs and on traces. The drill-down KQL in Step 6 should be `TraceId : "..."` on both sides — no field-name translation needed.

> [!WARNING]
> **`service.name` on `mule-logs` is `mule-container`, not the real app name.** This is a Mule SDK quirk: the resource detector for **logs** writes the generic value regardless of `mule.openTelemetry.exporter.resource.service.name`. **Traces** carry the real artifact ID. To join the two indexes on the same identity, use `artifact.id` (present on traces) and either replicate it onto logs via the Part 9 / 10 schema work, or rely on `Resource.service.namespace` and the `Resource.envId`/`workerId` Anypoint fields, which are correctly populated on both signals.

### Add a `duration_ms` runtime field on `Mule Traces`

The OTel Elasticsearch exporter writes `Duration` in **microseconds** for spans (a typical `mule:set-payload` lasts a few hundred microseconds — `651` is a representative value). We will alias it as `duration_ms` so every Lens visualization can use a sensible unit.

1. **Stack Management → Data Views → Mule Traces → Add field**.
2. Name: `duration_ms` · Type: `Double` · Set value: ON.
3. Painless script:

```painless
if (doc['Duration'].size() != 0) {
  emit(doc['Duration'].value / 1000.0);
}
```

4. Save.

A new computed field `duration_ms` is now usable in every Lens chart and KQL filter as if it were a real indexed field.

> [!IMPORTANT]
> **Confirm the `Duration` unit before trusting the runtime field.** Different OTel Collector + Elasticsearch exporter versions emit `Duration` in either nanoseconds or microseconds. Pick a sample document, look at a known-fast operation (e.g. `mule:set-payload` should be sub-millisecond), and divide by **1000** if you see values in the hundreds, or by **1,000,000** if you see values in the millions. The dashboard's hero metric is wrong if you mismatch the unit by 1000x — guess wrong and "slow traces" lists every span as 0.0001 ms.

---

## Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  Kibana — "Mule Observability" dashboard                            │
│                                                                     │
│   ┌────────────────────────┐  ┌──────────────────────────────────┐  │
│   │ Log volume by service  │  │ Error rate per service           │  │
│   │ (Lens · stacked area)  │  │ (Lens · line, COUNT/COUNT)       │  │
│   └────────────────────────┘  └──────────────────────────────────┘  │
│   ┌────────────────────────┐  ┌──────────────────────────────────┐  │
│   │ Span duration p95 by   │  │ Trace volume vs error spans      │  │
│   │ service (Lens · bar)   │  │ (Lens · line, two metrics)       │  │
│   └────────────────────────┘  └──────────────────────────────────┘  │
│   ┌─────────────────────────────────────────────────────────────┐   │
│   │ Slow traces — top 20 by total duration                      │   │
│   │ TraceId · service.name · root span name · duration_ms       │   │
│   │ ↳ click-through: open Discover on mule-logs filtered by     │   │
│   │   TraceId == clicked.TraceId                                │   │
│   └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐         ┌─────────────────────────────────────┐
│  Elasticsearch — mule-logs          │         │  Elasticsearch — mule-traces        │
│  (read by mule-logger / kibana user)│         │  (read by mule-tracer / kibana user)│
└─────────────────────────────────────┘         └─────────────────────────────────────┘
```

---

## Table of Contents

1. [Step 1 — Create the Empty Dashboard](#step-1--create-the-empty-dashboard)
2. [Step 2 — Panel 1: Log Volume by Service](#step-2--panel-1-log-volume-by-service)
3. [Step 3 — Panel 2: Error Rate per Service](#step-3--panel-2-error-rate-per-service)
4. [Step 4 — Panel 3: Span Duration p95 by Service](#step-4--panel-3-span-duration-p95-by-service)
5. [Step 5 — Panel 4: Trace Volume vs Error Spans](#step-5--panel-4-trace-volume-vs-error-spans)
6. [Step 6 — Panel 5: Slow Traces Table with Drill-Down](#step-6--panel-5-slow-traces-table-with-drill-down)
7. [Step 7 — Save, Export, Commit](#step-7--save-export-commit)
8. [Verification](#verification)
9. [Troubleshooting](#troubleshooting)

---

### Step 1 — Create the Empty Dashboard

1. **Analytics → Dashboard → Create dashboard**.
2. Title: `Mule Observability — Logs + Traces`.
3. Set the time filter to **Last 1 hour** so we can see the traffic we are about to drive.
4. Save the empty shell first — `Save → Save as new dashboard`. Saving early means each panel we add is incremental rather than at-risk if the browser tab closes.

> [!TIP]
> Drive a steady trickle of traffic against Part 7's App A in the background while we build, so every panel has fresh data:
>
> ```bash
> while true; do curl -s "https://hello-world-mule-direct-stream-ch2.<region>.cloudhub.io/hello" >/dev/null; sleep 2; done
> ```

> [!NOTE]
> *Screenshot — Empty Kibana dashboard titled "Mule Observability — Logs + Traces" with the Last 1 hour time picker.*

---

### Step 2 — Panel 1: Log Volume by Service

A stacked area showing how much each service is logging. First place to look when "is the app even running?" is in question.

1. **Add panel → Lens**.
2. **Data view:** `Mule Logs`.
3. Drag `@timestamp` onto the **Horizontal axis**. Confirm interval is `Auto`.
4. Drag the count metric (Records) onto **Vertical axis**.
5. Drag `Resource.service.name` onto **Breakdown**. Pick **Top values**, size 5.
6. Visualization type: **Area stacked**.
7. **Save and return**, panel title `Log volume by service`.

> [!NOTE]
> *Screenshot — Lens editor showing the area-stacked chart with `@timestamp` on the X axis, Records on the Y axis, and `Resource.service.name` as the breakdown.*

---

### Step 3 — Panel 2: Error Rate per Service

Errors as a fraction of total events, per service, per minute. The on-call screen.

1. **Add panel → Lens** · **Data view:** `Mule Logs`.
2. Horizontal axis: `@timestamp`.
3. Vertical axis — click the metric, switch to **Formula**:

```text
count(kql='SeverityText : "ERROR"') / count() * 100
```

   Set the metric label to `Error rate (%)`.
4. Breakdown: `Resource.service.name`, top values, size 5.
5. Visualization type: **Line**.
6. **Save and return**, panel title `Error rate per service (%)`.

> [!IMPORTANT]
> Why the explicit ratio formula instead of two separate metrics: a Lens line chart with two metrics renders two separate series, but for error-rate alerting the threshold lives on the ratio. Encoding it once here means future alerts can reuse the same KQL.

> [!NOTE]
> *Screenshot — Lens Formula editor showing `count(kql='SeverityText : "ERROR"') / count() * 100`, with the Y axis showing 0–100 %.*

---

### Step 4 — Panel 3: Span Duration p95 by Service

The 95th percentile of every span's duration, per service, over time. This is what catches latency creep before users do.

1. **Add panel → Lens** · **Data view:** `Mule Traces`.
2. Horizontal axis: `@timestamp`.
3. Vertical axis: drag `duration_ms` (the runtime field we created), function **Percentile**, percentile **95**.
4. Breakdown: `Resource.service.name`, top values, size 5.
5. Visualization type: **Bar vertical** (so quiet minutes do not silently drop the line).
6. **Save and return**, panel title `Span duration p95 (ms) by service`.

> [!TIP]
> p95 reveals the slow tail without being dragged around by a single outlier. Pair it with p99 in a tooltip-only metric if you want both numbers on hover; ship just p95 as the visible value.

---

### Step 5 — Panel 4: Trace Volume vs Error Spans

Two metrics on one line chart so the on-call engineer can eyeball "spike of traffic" against "spike of errors" in the same glance.

1. **Add panel → Lens** · **Data view:** `Mule Traces`.
2. Horizontal axis: `@timestamp`.
3. First metric: count of records, label `All spans`.
4. Second metric: **Formula** —

```text
count(kql='TraceStatus : 2')
```

   Label: `Error spans`. (Mule's traces use the numeric `TraceStatus` field — `0` unset, `1` OK, `2` ERROR — not the OTLP-string `StatusCode`.)
5. Visualization type: **Line**.
6. **Save and return**, panel title `Trace volume vs error spans`.

> [!NOTE]
> *Screenshot — line chart with two series, `All spans` and `Error spans`, plotted against the same time axis.*

---

### Step 6 — Panel 5: Slow Traces Table with Drill-Down

The hero panel: a sortable table of the slowest traces, where clicking a row jumps straight to the matching log lines.

#### 6.1 — Build the table

1. **Add panel → Lens** · **Data view:** `Mule Traces`.
2. Visualization type: **Table**.
3. Rows: `TraceId`, top values, size 20.
4. Optional second-level row: `Resource.service.name` (so we can see which service originated the root span).
5. Metric column 1: `Max` of `duration_ms` — call it `Total duration (ms)` (max of any span on the trace approximates the total wall time when the root span is present).
6. Metric column 2: `count()` — call it `Spans`.
7. Sort by `Total duration (ms)` descending.
8. **Save and return**, panel title `Slow traces — top 20 by duration`.

#### 6.2 — Wire the drill-down to logs

1. Click the panel's gear icon → **Create drilldown** → **Go to URL**.
2. **URL template:**

```text
/app/discover#/?_a=(index:'mule-logs',query:(language:kuery,query:'TraceId : "{{event.value}}"'),sort:!(!('@timestamp',asc)))&_g=(time:(from:now-24h,to:now))
```

3. **Trigger:** `Single click on a value`.
4. **Encode URL:** ON.
5. Save the drilldown.

> [!IMPORTANT]
> **The KQL field is `TraceId`, not `Attributes.trace.id`.** OTel Collector's Elasticsearch exporter (under `mapping.mode: raw`) writes the OTLP top-level `TraceId` directly to `_source.TraceId` on **both** logs and traces — so the same KQL works on both indexes. The MDC bridge from Part 5 also writes a parallel lowercase `trace_id` field; either path works in the drill-down. Earlier drafts that used `Attributes.trace.id` are wrong against the real document shape.

> [!IMPORTANT]
> The `{{event.value}}` placeholder is replaced with the clicked cell's value. Configure the drilldown on the **`TraceId` column** (Lens lets you scope a drilldown to a specific column). Configuring it on a numeric column will pass the duration string into the KQL — produces zero hits and looks broken.

> [!TIP]
> Replace `mule-logs` in the URL with the **data view ID** for `Mule Logs` (looks like `f1e2d3c4-…`) for a more stable link. Find the ID in **Stack Management → Data Views → Mule Logs**, in the URL of the data view edit page. Names work today; IDs survive a rename.

> [!NOTE]
> *Screenshot — the drilldown configuration dialog showing the URL template and the `TraceId` column scope, then a follow-up screenshot of the resulting Discover session filtered by a single `TraceId`.*

---

### Step 7 — Save, Export, Commit

Final save of the dashboard:

1. **Save** → confirm title `Mule Observability — Logs + Traces`.
2. **Set Time Filter** → check the box so the dashboard remembers `Last 1 hour` as its default time range.

#### Export the saved objects as NDJSON

Kibana ships everything we built as a portable NDJSON file.

1. **Stack Management → Saved Objects**.
2. Filter by **Type:** `dashboard`, `lens`, `index-pattern`.
3. Tick:
   - The **Mule Observability — Logs + Traces** dashboard.
   - All five Lens visualizations.
   - The `Mule Logs` and `Mule Traces` data views.
4. Click **Export** → enable **Include related objects** → save as `mule-observability-dashboard.ndjson`.

> [!IMPORTANT]
> **Include related objects** is what bundles the data views and the runtime field along with the dashboard. Without it, importing into a fresh cluster fails with `index pattern not found` because the dashboard references data views that do not exist there.

#### Commit the NDJSON to the series repo

Place the file at:

```text
./assets/kibana/mule-observability-dashboard.ndjson
```

📄 Asset link in the post: [./assets/kibana/mule-observability-dashboard.ndjson](./assets/kibana/mule-observability-dashboard.ndjson)

Future readers — or our own future clusters — import it via **Stack Management → Saved Objects → Import**.

> [!NOTE]
> *Screenshot — the Saved Objects export dialog with the dashboard, lens visualizations, and data views ticked, and the "Include related objects" toggle ON.*

---

## Verification

Three checks to confirm the dashboard is doing what we asked of it.

**1. Every panel shows data over the last hour.**

Open the dashboard with the time filter at `Last 1 hour`. All five panels should render non-empty:

- Log volume should show stacked bands per `service.name`.
- Error rate should be flat near 0% in the demo (we have no failure flow yet); inject a controlled failure with `curl -s http://localhost:8081/hello?fail=1` if you want to see it move.
- Span duration p95 should be in the tens to low hundreds of milliseconds for the hello-world apps.
- Trace volume should track the request loop's rate; error spans line should be flat at 0.
- Slow traces table should list 20 traces sorted by `Total duration (ms)`.

**2. The drill-down hops indexes correctly.**

Click any row of the Slow Traces table. We expect:

- A new browser tab on Discover.
- Data view set to `Mule Logs`.
- The KQL bar pre-filled with `Attributes.trace.id : "<long hex>"`.
- The result is the **same** log lines we would get by manually filtering `mule-logs` for that `trace.id` — the count should match the count from Part 7's correlation tests.

**3. The NDJSON export imports cleanly into a fresh cluster.**

The portable test: fire up a throwaway Kibana, import `mule-observability-dashboard.ndjson` via **Saved Objects → Import**, and open the dashboard. If the data views resolve, the dashboard renders panels (with whatever data the throwaway cluster has), and the drill-down still works, the export is portable.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Panel renders "No results" but the underlying index has documents | Data view's time field is not `@timestamp` | **Stack Management → Data Views → Mule Logs / Mule Traces** → set time field to `@timestamp` |
| `duration_ms` runtime field returns `null` for every row | OTel Collector emits `Duration` under a nested path on this exporter version, e.g. `attributes.duration` | In the runtime field, change `doc['Duration']` to the actual path; preview a sample document to find it |
| Drill-down lands on Discover but with zero hits | The `mule-logs` records do not carry `Attributes.trace.id` because `mule.put.trace.id.and.span.id.in.mdc=true` is missing on one of the apps | Re-check Part 5 / Part 7 Step 4 — the property must be set on **every** app whose logs we want correlated |
| Drill-down URL contains `{{event.value}}` literal instead of the trace ID | `Encode URL` toggle is off, or the drilldown was attached to the panel rather than the column | Re-create the drilldown with the column-level scope (Step 6.2) and `Encode URL` ON |
| Imported NDJSON fails with `Could not find data view` | Export did not include related objects | Re-export with **Include related objects** ON; re-import |
| Error rate panel shows `NaN` when there is no traffic | `count() = 0`, division-by-zero | Acceptable — Lens renders gaps; if the dashboard exporter requires a number, wrap the formula: `count(kql='...') / max(0.0001, count()) * 100` |
| Panel shows two `service.name` slices that look identical (e.g. `mule-container` and the real name) | RTF and CH2 sometimes default `service.name` to `mule-container` if the app's `mule.openTelemetry.exporter.resource.service.name` property is missing | Set the property on every deployment; rebuild the dashboard's breakdown after the fix and the duplicate will disappear |
| Total duration in the Slow Traces table is suspiciously high | `max(duration_ms)` is the slowest *span*, not the full trace duration. For multi-service traces with sequential calls, span max ≈ root span ≈ trace duration; for parallel calls it is an underestimate | Acceptable for the demo; for production-grade trace duration, query the root span only (`ParentSpanId : "" `) and use its `duration_ms` directly |

---

## What We Covered

- We built **five Lens panels** on the same dashboard — log volume, error rate, span duration p95, trace volume vs error spans, and a slow-traces table — driven by both `mule-logs` and `mule-traces`.
- We added a **`duration_ms` runtime field** so traces visualize in milliseconds, not nanoseconds.
- We wired a **column-level drilldown** from the slow-traces table to a Discover session filtered by `Attributes.trace.id`, so a click on a slow trace jumps straight to its log timeline.
- We exported the dashboard, the five Lens visualizations, and both data views as a single **NDJSON** file with `Include related objects` enabled, and committed it to the series repo for one-click import on any future cluster.
- We turned the correlation that has been theoretical since Part 5 into something **visible** — and clickable.

We now have the whole stack stood up, instrumented, and observable end-to-end:

1. **Elasticsearch + Kibana** on AWS with HTTP-only Basic Auth (Parts 1–2).
2. **Indexes, roles, and least-privilege users** for logs and traces (Part 3).
3. **Two log-shipping paths** — Log4j2 HTTP appender (Part 4) and Direct Telemetry Stream (Part 5) — with the conceptual deep-dive on why OTLP needs a translator in front of Elastic (Part 6).
4. **Distributed traces** between two Mule 4.11 apps via OTLP → OTel Collector → Elasticsearch, with `trace.id` correlation already in the log records (Part 7).
5. **A unified Kibana dashboard** that pulls all of it onto one screen (this post).

> ➡️ **Coming next — a follow-up series.** *"From OTel Collector to Elastic APM"* will swap the OTel Collector + plain `mule-traces` index for **Elastic APM Server**, which natively ingests OTLP, writes to the APM data streams, and unlocks Kibana's **Observability → APM** UI — service maps, latency distributions, error groupings, span flame graphs. Same Mule properties (`mule.openTelemetry.tracer.exporter.endpoint` just changes), one fewer component to operate, deeper Kibana-native experience.

Thanks for following along through the whole series — the cluster, the wiring, and the screen. Now we have the foundation to layer alerts, dashboards per app, ECS-native indexes, ILM retention, and APM Server on top, depending on where the work takes us next.

---

## References

- [Lens — Kibana Docs](https://www.elastic.co/guide/en/kibana/current/lens.html)
- [Runtime fields — Elasticsearch Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/runtime.html)
- [Painless scripting language — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/painless/current/index.html)
- [Dashboard drilldowns — Kibana Docs](https://www.elastic.co/guide/en/kibana/current/drilldowns.html)
- [Saved objects export and import — Kibana Docs](https://www.elastic.co/guide/en/kibana/current/managing-saved-objects.html)
- [KQL — Kibana Query Language](https://www.elastic.co/guide/en/kibana/current/kuery-query.html)
- [Elasticsearch Exporter — OTel Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/elasticsearchexporter)
- [Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/index.html)
