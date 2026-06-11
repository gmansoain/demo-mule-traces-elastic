---
title: "Operating the Stack — KQL Recipes, Alerts, and Dashboards — Part 12"
slug: "mule-elastic-kql-alerts-dashboards"
description: "Concrete KQL queries, alert rules, and dashboards for the ECS-shaped Mule logs and traces from Parts 9–11. Banking-scenario examples, copy-pasteable, against the real field names."
author: "Gonzalo Marcos"
date: 2026-06-10
status: not validated
lang: en
category: observability
tags:
  - kibana
  - elasticsearch
  - logging
  - alerting
  - kpis
  - architecture-diagram
  - english
  - series
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 12
type: tutorial
difficulty: intermediate
read_time: 17
mule_version: "4.11"
platform:
  - anypoint-platform
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 12](https://img.shields.io/badge/Part-12-16a085) ![17 min](https://img.shields.io/badge/Read_Time-17_min-lightgrey)

# Operating the Stack — KQL Recipes, Alerts, and Dashboards — Part 12

Across [Parts 1–11](#) we built the entire pipeline: Elasticsearch and Kibana on AWS, two least-privilege users, two Mule 4.11 apps shipping logs and traces via Direct Telemetry Stream, an OpenTelemetry Collector reshaping OTLP into ECS-native JSON, and a unified dashboard that joins the two indexes on `trace.id`. The cluster is *full of data*. Now the question is: **how do we use it?**

This post is the operational complement to the rest of the series. We will walk through three things, all concrete and copy-pasteable against the shape we get from Parts 9–11:

- **KQL queries** in three tiers — the single-field filters that go on muscle memory, the multi-clause production queries that answer real on-call questions, and the operational queries that catch deploy drift and dropouts before users complain.
- **Alert rules** that page the right person at the right threshold — error rate, latency p95, traffic dropouts, version drift, calling-client failures.
- **Dashboards** organized by audience — on-call, app-team, executive, deploy/audit — each with concrete panel recipes, not generic templates.

Every example uses banking-scenario values consistent with Parts 9–11 (`process-payments-sepa`, `retail-banking`, `business.domain: payments`, `client_id: Mobile App`) and the ECS-shaped indexes (`mule-logs-ecs`, `mule-traces-ecs`) we land on after Part 11. Earlier OTLP-shaped data (`mule-logs`, `mule-traces`) uses different field names — there is a translation table at the bottom of the post.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 11 — Reshaping OTLP Logs into ECS-Native JSON](#) · **Next:** Series wrap-up

> [!WARNING]
> **HTTP-only, demo-grade.** Same posture as the rest of the series. Alerts that page on-call need a real notification channel (PagerDuty, Slack, Opsgenie); the demo cluster does not have one wired up. Treat the alert *conditions* below as production-shaped; the *connectors* are an exercise for the reader.

---

## What We Will Cover

- A field-shape cheat sheet for `mule-logs-ecs` vs `mule-traces-ecs` so every KQL example below is unambiguous.
- **Section A** — twenty-five KQL recipes, three tiers, indexed by the question they answer.
- **Section B** — eight alert rules, complete with KQL conditions, threshold values, and grouping.
- **Section C** — four dashboards, panel-by-panel, with the data view and Lens config each one needs.
- **Section D** — the cross-index drilldown pattern that lets one click on a slow trace open the matching log lines.
- A translation table for readers still on Parts 5–8's OTLP-shaped indexes.

---

## Prerequisites

- The full pipeline from [Parts 9–11](#): Mule apps emitting OTLP with the eleven-field schema, OTel Collector reshaping into ECS, indexes named `mule-logs-ecs` and `mule-traces-ecs`.
- Two Kibana data views, one per index, with `@timestamp` set as the time field.
- A Kibana user with `kibana_admin` for saving objects and creating alert rules. The `mule-logger` and `mule-tracer` write-only users cannot save anything.
- For alerts: an external connector (Slack / Email / PagerDuty / Webhook) configured under **Stack Management → Connectors**. The KQL conditions below work without it; the page-the-on-call wiring needs it.

---

## Field-Shape Cheat Sheet

Every KQL example below references the field paths in this table. The two indexes have **different naming conventions**, so the same concept needs different syntax depending on which index the query runs against.

| Concept | `mule-logs-ecs` | `mule-traces-ecs` |
| --- | --- | --- |
| Service identity | `service.name`, `service.namespace`, `service.version`, `service.environment`, `service.node.name` | `Resource.service.name`, `Resource.service.namespace`, `Resource.service.version`, `Resource.service.instance.id` |
| Deployment | `deployment.region`, `deployment.target` | `Resource.deployment.environment`, `Resource.deployment.region`, `Resource.deployment.target` |
| API / domain | `api.layer`, `business.domain` | `Resource.api.layer`, `Resource.business.domain` |
| Severity / status | `log.level` (`INFO`/`ERROR`/...), `event.severity` | `TraceStatus` (numeric: `0` unset, `1` OK, `2` ERROR), `Kind` |
| Body / span name | `message` | `Name` |
| Trace ID | `trace.id` (and duplicate flat `trace_id`) | `TraceId` |
| Span ID | `span.id` (and duplicate flat `span_id`) | `SpanId`, `ParentSpanId` |
| Correlation ID | `labels.correlation_id` | `Attributes.labels.correlation_id` |
| Calling client | `client_id` | `Attributes.client_id` |
| HTTP route, method | `http.route`, `http_method` | `Attributes.http.route`, `Attributes.http_method` |
| Mule flow location | `mule.flow.processor_path` | `Attributes.mule.flow.processor_path` |
| Anypoint platform IDs | `labels.anypoint_env_id`, `labels.anypoint_org_id`, `labels.anypoint_root_org_id` | `Resource.envId`, `Resource.orgId`, `Resource.rootOrgId` |
| Worker / pod / replica | `host.id`, `service.node.name` | `Resource.workerId`, `Resource.service.instance.id` |
| Duration | n/a | `Duration` (microseconds — divide by `1000` for ms) |
| Span timestamps | `@timestamp` only | `@timestamp` (start) + `EndTimestamp` (end) |

> [!IMPORTANT]
> **Field-name asymmetry is a tax we pay on every cross-index query.** A panel that sums error counts from logs and error spans from traces, grouped by service, has to reference *two different* `service.name` paths. Either keep that translation in your head, or use **runtime fields** on the data views to alias one to the other (e.g., a runtime field on `mule-traces-ecs` named `service.name` that emits `Resource.service.name` — Kibana's autocomplete then sees the same name on both indexes).

---

## Section A — KQL Recipes

Three tiers. Tier 1 is what you should be able to type without looking. Tier 2 is the muscle of the on-call kit. Tier 3 is what catches problems before users notice.

### Tier 1 — Single-field filters (muscle memory)

| #      | Index             | KQL                                               | What it answers                                                                   |
| ------ | ----------------- | ------------------------------------------------- | --------------------------------------------------------------------------------- |
| **1**  | `mule-logs-ecs`   | `log.level : "ERROR"`                             | All error log lines, any service, any time.                                       |
| **2**  | `mule-logs-ecs`   | `service.environment : "prod"`                    | Only production traffic.                                                          |
| **3**  | `mule-logs-ecs`   | `service.name : "process-payments-sepa"`          | All logs from one service.                                                        |
| **4**  | `mule-logs-ecs`   | `business.domain : "payments"`                    | Everything the payments domain produced (across SEPA, SWIFT, etc.).               |
| **5**  | `mule-logs-ecs`   | `client_id : "Mobile App"`                        | Every line from requests originating in the mobile client.                        |
| **6**  | `mule-logs-ecs`   | `http.route : "/hello"`                           | Logs from a single route.                                                         |
| **7**  | `mule-traces-ecs` | `Resource.service.name : "process-payments-sepa"` | All spans from one service.                                                       |
| **8**  | `mule-traces-ecs` | `TraceStatus : 2`                                 | Only error spans.                                                                 |
| **9**  | `mule-traces-ecs` | `Kind : "SPAN_KIND_SERVER"`                       | Only inbound listener spans (request boundaries — useful for end-to-end latency). |
| **10** | `mule-traces-ecs` | `not ParentSpanId : *`                            | Only **root** spans (the parent of the trace tree, aka the entry point).          |

### Tier 2 — Multi-clause production queries

| # | Index | KQL | What it answers |
| --- | --- | --- | --- |
| **11** | `mule-logs-ecs` | `service.environment : "prod" and business.domain : "payments" and log.level : "ERROR"` | Production payments errors. The default first-30-seconds-of-an-incident filter. |
| **12** | `mule-logs-ecs` | `service.name : "process-payments-sepa" and not client_id : "Mobile App"` | Errors on SEPA from non-mobile callers. Splits incidents that affect one channel from incidents that affect the service itself. |
| **13** | `mule-logs-ecs` | `mule.flow.processor_path : "hello-gon-subflow/processors/4"` | Every line emitted from a specific Logger component, identified by its flow path. Useful when a runbook says "look for log line X right after step Y." |
| **14** | `mule-logs-ecs` | `service.environment : "prod" and host.id : "64d8569b7d-wrk79"` | All logs from one specific replica. Catches *"is this pod misbehaving?"*. |
| **15** | `mule-logs-ecs` | `trace.id : "535b3a368e16c5ae457557c55b1465cb"` | Every log line from one specific request — the canonical "what happened to *that* request?" query. |
| **16** | `mule-traces-ecs` | `TraceId : "535b3a368e16c5ae457557c55b1465cb"` | Every span from one specific request — paired with #15 above to follow the request end-to-end. |
| **17** | `mule-traces-ecs` | `Resource.deployment.environment : "prod" and Resource.api.layer : "process" and Duration > 50000` | Production process-layer spans slower than 50 ms. Quick win for *"which calls into the system layer are dragging?"*. |
| **18** | `mule-traces-ecs` | `Resource.service.name : "process-payments-sepa" and Kind : "SPAN_KIND_SERVER" and Duration > 1500000` | Inbound SEPA requests slower than 1.5 s — wall-clock latency above SLO. |
| **19** | `mule-traces-ecs` | `Resource.business.domain : "payments" and TraceStatus : 2` | Failed spans across the payments domain — for both SEPA and SWIFT services. |
| **20** | `mule-logs-ecs` | `labels.correlation_id : "0a012400-64d3-11f1-8317-420eb69acb4d"` | Every line that shares this business correlation ID — useful when the trace ID has rotated mid-flight (async retries) but the business ID survives. |

### Tier 3 — Operational queries (deploy drift, capacity, dropouts)

| # | Index | KQL | What it answers |
| --- | --- | --- | --- |
| **21** | `mule-logs-ecs` | `service.environment : "prod" and not service.version : "1.2.0"` | Production replicas not on the expected version. Surfaces stuck rolling deploys. |
| **22** | `mule-logs-ecs` | `service.environment : "prod" and not mule.runtime.version : "4.12.0"` | Production replicas on a non-target Mule runtime version. Catches a missed runtime patch. |
| **23** | `mule-traces-ecs` | `Resource.service.name : "process-payments-sepa" and not Resource.service.version : "1.2.0"` | Same as #21 but on traces — useful when the deploy is so broken the app can't log but the OTel SDK still emits a startup span. |
| **24** | `mule-logs-ecs` | `service.name : "process-payments-sepa" and exists : "trace.id"` | Logs that *do* carry a trace context. Inverting it (`not exists : "trace.id"`) finds startup, scheduled, and async lines that have no trace — which is correct, but the count should be small in steady-state production. |
| **25** | `mule-logs-ecs` | `service.environment : "prod" and labels.anypoint_root_org_id : "37aa8fe0-4188-44c1-a9b5-bdce3ca12f0b"` | Filter to a specific Anypoint root org — for multi-tenant orgs that share a Kibana cluster. |

> [!TIP]
> **Save the queries you use more than twice.** Kibana's "Save" button on the search bar drops them into the Discover saved-searches list, which becomes the team's shared shortcut menu. The list is what new on-call rotates pin to their browser bookmarks.

---

## Section B — Alert Rules

Eight production-ready alerts. Each one is a Kibana Alerting rule (**Stack Management → Rules and Connectors → Create rule**) — most are **Threshold** or **Query** rule types, both of which read KQL natively.

> [!IMPORTANT]
> **Rule types matter.** Use **Threshold** when the alert fires on a *count over time* (e.g., >5 errors per minute). Use **Query** (or **Custom Threshold** in newer Kibana) when the condition is a KQL expression that returns rows. **Ratio** rules are useful for percentages but are not always present in older Kibana — substitute with two threshold rules combined in an action where needed.

### B1 — Per-service error rate spike

| Field | Value |
| --- | --- |
| Rule type | Threshold |
| Index | `mule-logs-ecs` |
| KQL | `log.level : "ERROR" and service.environment : "prod"` |
| Aggregation | `count()` per minute |
| Group by | `service.name` |
| Threshold | `> 5` for `5` consecutive minutes |
| Action | Slack channel `#oncall-platform`, with the offending `service.name` and the count. |

Catches the typical incident: a service starts throwing errors at a rate well above its background.

### B2 — Latency p95 per route

| Field | Value |
| --- | --- |
| Rule type | Threshold (or **Custom threshold** if available) |
| Index | `mule-traces-ecs` |
| KQL | `Resource.deployment.environment : "prod" and Kind : "SPAN_KIND_SERVER"` |
| Aggregation | 95th percentile of `Duration / 1000` (ms) over `10` minutes |
| Group by | `Attributes.http.route` |
| Threshold | `> 1500` ms |

Pages on-call when a single route slows down. Filtering to `Kind: SPAN_KIND_SERVER` ensures we measure the inbound-listener span (full request wall-clock) and not internal processor spans.

> [!TIP]
> Use a Lens runtime field `duration_ms` (Painless: `emit(doc['Duration'].value / 1000)`) on `mule-traces-ecs` so the rule can reference `duration_ms` directly without inline arithmetic. Same trick we used in Part 8.

### B3 — Trace volume drop (silence detection)

| Field | Value |
| --- | --- |
| Rule type | Threshold |
| Index | `mule-traces-ecs` |
| KQL | `Resource.service.name : "process-payments-sepa" and Resource.deployment.environment : "prod"` |
| Aggregation | `count()` over `5` minutes |
| Threshold | `< 1` (i.e. zero spans in the last 5 minutes) |
| Quiet hours | Suppress between 02:00–05:00 if traffic is genuinely zero overnight. |

Catches a downed app, a downed tracer exporter, or a downed collector — all three look like "no spans" downstream. Mirror the rule on `mule-logs-ecs` for symmetric coverage.

### B4 — Service stops logging in business hours

| Field | Value |
| --- | --- |
| Rule type | Threshold |
| Index | `mule-logs-ecs` |
| KQL | `service.name : "process-payments-sepa" and service.environment : "prod"` |
| Aggregation | `count()` over `5` minutes |
| Threshold | `< 1` |
| Schedule | Every minute, only between 08:00–20:00 weekdays (rule schedule, not condition). |

Different from B3 — this one fires when the **app's logs** disappear, which is an earlier signal than its traces (the OTel SDK keeps emitting heartbeats even when the app is wedged).

### B5 — Failure rate by calling client

| Field | Value |
| --- | --- |
| Rule type | Custom Threshold (or two Threshold rules + an action that compares them) |
| Index | `mule-logs-ecs` |
| Numerator KQL | `log.level : "ERROR" and service.environment : "prod"` |
| Denominator KQL | `service.environment : "prod"` |
| Aggregation | `count(ERROR) / count(all) * 100` over `15` minutes |
| Group by | `client_id` |
| Threshold | `> 10` % |

Catches a misbehaving caller (bad payloads, expired token, broken integration on the client side). Grouping by `client_id` is what makes it actionable — paging the platform on-call when a single bad client is responsible for 40% of errors lets the right team take over.

### B6 — Span error count for a critical service

| Field | Value |
| --- | --- |
| Rule type | Threshold |
| Index | `mule-traces-ecs` |
| KQL | `Resource.service.name : "process-payments-sepa" and TraceStatus : 2` |
| Aggregation | `count()` over `5` minutes |
| Threshold | `> 0` |

Server-side errors on the most critical service. Threshold is intentionally low — if SEPA payments throws *any* span error in production, page someone.

### B7 — Cross-replica version drift

| Field | Value |
| --- | --- |
| Rule type | Custom Threshold (cardinality) |
| Index | `mule-logs-ecs` |
| KQL | `service.environment : "prod"` |
| Aggregation | `cardinality(service.version)` over `10` minutes |
| Group by | `service.name` |
| Threshold | `> 1` |

A healthy rolling deploy briefly reports two versions per service while replicas are being replaced. A *stuck* deploy keeps reporting two versions for hours. Setting the window to 10 minutes catches the latter without paging on every successful deploy.

### B8 — Mismatched Anypoint environment

| Field | Value |
| --- | --- |
| Rule type | Query |
| Index | `mule-logs-ecs` |
| KQL | `service.environment : "prod" and not labels.anypoint_env_id : "<expected-prod-env-uuid>"` |
| Threshold | `> 0` results in any 5-minute window |

Catches the rare-but-real failure mode: a `prod`-tagged Mule app is actually running in a non-prod Anypoint environment because of a Maven deployment mistake. The `labels.anypoint_env_id` is auto-injected by Mule's OTel SDK; we own the expected value, so any mismatch is a misconfigured deploy.

---

## Section C — Dashboards

Four dashboards, organized by audience. Each is a separate Kibana dashboard (NDJSON-exportable per Part 8) with a defined purpose and time-filter default.

### C1 — On-call dashboard ("what is broken right now?")

**Audience:** the engineer paged at 3 AM. **Time default:** Last 30 minutes. **Layout:** 2×3 grid.

| Panel | Index | Lens config |
| --- | --- | --- |
| Open alerts | n/a | Embed of the Kibana **Alerts** flyout filtered to `mule-*` rules. The dashboard's headline. |
| Error log volume per service | `mule-logs-ecs` | Stacked area, X = `@timestamp`, Y = `count()`, Breakdown = `service.name` (top 5), filter `log.level : "ERROR"`. |
| Latest 50 ERROR logs | `mule-logs-ecs` | Table, columns = `@timestamp`, `service.name`, `client_id`, `message`. KQL filter `log.level : "ERROR"`. Sorted by `@timestamp` desc. |
| Span error count by service | `mule-traces-ecs` | Bar vertical, X = `Resource.service.name` (top 10), Y = `count()`, KQL `TraceStatus : 2`. |
| Latency p95 by route (last 30 min) | `mule-traces-ecs` | Line, X = `@timestamp`, Y = `percentile(Duration, 95) / 1000` (ms), Breakdown = `Attributes.http.route` top 5, KQL `Kind : "SPAN_KIND_SERVER"`. |
| Trace volume vs error spans | `mule-traces-ecs` | Line with two series: `count()` (label "All spans"), `count(kql='TraceStatus : 2')` (label "Error spans"). |

### C2 — App-team dashboard ("how is *my* service?")

**Audience:** the engineer who just pushed a change. **Time default:** Last 1 hour. **Filter control at the top:** dropdown bound to `service.name` so the team picks which service the dashboard scopes to.

| Panel | Index | Lens config |
| --- | --- | --- |
| Log volume by `log.level` | `mule-logs-ecs` | Stacked area, X = `@timestamp`, Y = `count()`, Breakdown = `log.level`. Honors the dashboard's `service.name` filter. |
| p50 / p95 / p99 latency | `mule-traces-ecs` | Line, three Y series: `percentile(Duration, 50) / 1000`, `percentile(Duration, 95) / 1000`, `percentile(Duration, 99) / 1000`. KQL `Kind : "SPAN_KIND_SERVER"`. |
| Span tree — slowest 20 spans | `mule-traces-ecs` | Table, rows = `Name` + `Attributes.mule.flow.processor_path`, metric = `max(Duration) / 1000`, sorted desc. |
| Top callers by `client_id` | `mule-logs-ecs` | Bar horizontal, rows = `client_id` (top 10), metric = `count()`. |
| Recent slow traces | `mule-traces-ecs` | Table, columns = `@timestamp`, `TraceId`, `Name`, `Duration`, `Attributes.client_id`. KQL `Kind : "SPAN_KIND_SERVER" and Duration > 500000`. Limit 20. |
| Active replicas | `mule-logs-ecs` | Table, rows = `host.id`, metrics = `cardinality(trace.id)` (request count) and last `@timestamp` (heartbeat). |

### C3 — Cross-LOB executive dashboard ("how is the platform?")

**Audience:** anyone who wants the 30-second answer to *"is the platform OK?"*. **Time default:** Last 24 hours. **Layout:** 2×2 grid plus a top KPI strip.

| Panel | Index | Lens config |
| --- | --- | --- |
| Top KPIs (4 cards) | `mule-logs-ecs` and `mule-traces-ecs` | Four metric tiles — total request volume (count of root spans on traces), error rate (% of `log.level: ERROR` over total logs), p95 latency (server spans, ms), distinct active services (`cardinality(service.name)`). |
| Request volume by `business.domain` | `mule-traces-ecs` | Donut chart, slice = `Resource.business.domain`, metric = `count(kql='Kind : "SPAN_KIND_SERVER"')`. Click → drill to a domain-filtered version of C2. |
| Success rate per `service.namespace` | `mule-logs-ecs` | Gauge per namespace, formula = `1 - (count(kql='log.level : "ERROR"') / count())`, threshold bands at 99% / 99.9%. |
| Latency p95 trend per `api.layer` | `mule-traces-ecs` | Line over 7 days (override the dashboard time filter on this panel), X = `@timestamp`, Y = `percentile(Duration, 95) / 1000`, Breakdown = `Resource.api.layer`. |
| Top-5 noisiest services by log volume | `mule-logs-ecs` | Bar horizontal, rows = `service.name` (top 5), metric = `count()`. Surfaces a spammy service before it costs us in storage. |

### C4 — Audit / deploy dashboard ("what is running where?")

**Audience:** release manager, on-call during a rolling deploy. **Time default:** Last 6 hours.

| Panel | Index | Lens config |
| --- | --- | --- |
| Versions in production right now | `mule-logs-ecs` | Table, rows = `service.name`, columns = distinct values of `service.version`, metric = `cardinality(host.id)` (replicas on each version). KQL filter `service.environment : "prod"`. |
| Mule runtime distribution | `mule-logs-ecs` | Bar vertical, X = `mule.runtime.version`, Y = `cardinality(service.name)`. Catches stragglers on an old runtime patch. |
| Replica heartbeat | `mule-logs-ecs` | Table, rows = `host.id`, columns = `service.name`, last seen = `max(@timestamp)`. Pods that stopped emitting are visible at a glance. |
| Anypoint org distribution | `mule-logs-ecs` | Bar horizontal, rows = `labels.anypoint_env_id`, metric = `cardinality(service.name)`. Sanity-check before a multi-env promotion. |

### C5 — Operational analytics dashboard ("what is *trending* on the platform?")

**Audience:** the engineer doing weekly review or trend analysis — not paged, but reading the platform's pulse. **Time default:** Last 24 hours, comparable across days. **Layout:** ten panels covering the most useful operational angles, each one a Lens visualization. The full step-by-step Kibana-UI walkthrough lives in [`kibana_dashboards.md`](./kibana_dashboards.md) — the table below is the panel inventory.

| Panel | Index | Lens config |
| --- | --- | --- |
| Error rate (%) by service | `mule-logs-ecs` | Line. X = `@timestamp`, Y = Formula `count(kql='log.level : "ERROR"') / count() * 100`, Breakdown = `service.name` top 10. KQL `service.environment : "prod"`. |
| Errors over time by `api.layer` | `mule-logs-ecs` | Stacked area. X = `@timestamp`, Y = `count()`, Breakdown = `api.layer` (top 3). KQL `service.environment : "prod" and log.level : "ERROR"`. |
| Slow transactions — top 50 | `mule-traces-ecs` | Table sorted by `max(duration_ms)` desc. Rows = `TraceId`, `Resource.service.name`, `Attributes.http.route`. Wired with a column-level drilldown on `TraceId` to Discover on `mule-logs-ecs`. KQL `Resource.deployment.environment : "prod" and Kind : "SPAN_KIND_SERVER" and Duration > 1500000`. |
| Requests per service over time (TPS trend) | `mule-traces-ecs` | Line. X = `@timestamp`, Y = `count()`, Breakdown = `Resource.service.name` top 5. KQL filters to `Kind : "SPAN_KIND_SERVER"` so we count request boundaries, not internal spans. |
| Load by replica (one app) | `mule-traces-ecs` | Line. X = `@timestamp`, Y = `count()`, Breakdown = `Resource.workerId` top 10. KQL filters to one `Resource.service.name`. |
| Active replicas (now) | `mule-traces-ecs` | Metric tile. Primary metric = `cardinality(Resource.workerId)`. Drops when a deploy runs short on healthy pods. |
| Errors by `service.version` | `mule-logs-ecs` | Bar vertical. X = `service.version`, Y = `count()`, Breakdown = `service.name` top 5. KQL `log.level : "ERROR" and service.environment : "prod"`. Pairs with #1 to surface deploy regressions. |
| Error rate by flow processor | `mule-logs-ecs` | Bar horizontal. Rows = `mule.flow.processor_path` top 20. Three metrics: `count(kql='log.level : "ERROR"')`, `count()`, Formula `count(kql='log.level : "ERROR"') / count() * 100`. Sorted by error-rate desc. |
| Top consumers by `client_id` | `mule-logs-ecs` | Bar horizontal. Rows = `client_id` top 20. Metrics: `cardinality(trace.id)` (request count, dedupes log spam), `count()` (log volume). |
| Client error rate | `mule-logs-ecs` | Table sorted by Formula `count(kql='log.level : "ERROR"') / count() * 100` desc. Rows = `client_id` top 20 with `Min documents per term: 100`. Metrics = errors, total, error rate %. |
| Thread saturation (3 sub-panels) | `mule-logs-ecs` | (a) Line, Y = `cardinality(process.thread.id)`, Breakdown = `service.name`. (b) Line, Y = Formula `count() / unique_count(process.thread.id)`. (c) Table sorted by `count()` desc, rows = `process.thread.name`, last 5 minutes. Optional 4th panel breaks down by `mule.thread.tier` runtime field. |

> [!IMPORTANT]
> Two **runtime fields** are prerequisites for this dashboard. Create them once on the data view (Stack Management → Data views → Add field):
>
> - On `Mule Traces`: `duration_ms` of type `Double`, script `if (doc['Duration'].size() != 0) { emit(doc['Duration'].value / 1000.0); }` — converts microseconds to milliseconds for #3, #4, #5, #11.
> - On `Mule Logs`: `mule.thread.tier` of type `Keyword`, script extracts `CPU_LITE` / `CPU_INTENSIVE` / `IO` from `process.thread.name.keyword`. Used by panel 11d.
>
> Without `duration_ms`, every duration metric needs `Duration / 1000` inline (works, but verbose).

---

## Section D — Cross-Index Drilldown Pattern

The cross-index correlation we built in Part 8 uses one click to jump from a slow trace in `mule-traces-ecs` to its log lines in `mule-logs-ecs`. The mechanism is a Lens **column-level URL drilldown**. The wrinkle for the ECS-shaped indexes: the trace ID field has *different names* on the two indexes — `TraceId` on traces, `trace.id` on logs.

Configure the drilldown on the `TraceId` column of any traces table panel:

```text
URL template:
/app/discover#/?_a=(index:'mule-logs-ecs',query:(language:kuery,query:'trace.id : "{{event.value}}"'),sort:!(!('@timestamp',asc)))&_g=(time:(from:now-24h,to:now))

Trigger: Single click on a value
Encode URL: ON
```

A click on a slow trace row opens a Discover session on `mule-logs-ecs`, pre-filtered to every log line that shares the `trace.id`. The chronological sort surfaces the *order of events* during that request — usually the fastest path from "this request was slow" to "this is what it did at each step."

> [!IMPORTANT]
> **Replace the data view name with the data view ID** for production stability. Names work; IDs survive a rename. Look up the ID in **Stack Management → Data Views**, in the URL of the data view edit page (looks like `f1e2d3c4-...`).

> [!TIP]
> Mirror the drilldown on the **logs-side panels** too: when on-call clicks an ERROR log line, jump to the trace document for that request. URL template flips the field names: `index:'mule-traces-ecs', query:'TraceId : "{{event.value}}"'`. Do not require people to remember which direction they are going — make both clicks land somewhere useful.

---

## Verification

Three checks to confirm the queries, alerts, and dashboards are wired against the real data.

**1. Every Tier 1 KQL returns hits.**

Drive light traffic to each service for ~5 minutes, then paste each Tier 1 query into Discover with a Last-15-minutes time filter. Every query should return at least one hit. A query that returns zero usually means a field-name typo — Lens / Discover autocomplete catches that.

**2. Each alert preview matches expected behavior.**

In Kibana → **Stack Management → Rules and Connectors**, open each alert, click **Test** → **Preview**. The preview shows what the rule would have fired in the last hour. Tune thresholds until the preview matches the on-call cadence the team actually wants — too noisy and people mute it, too quiet and it misses the incident.

**3. Each dashboard panel renders non-empty.**

Open every dashboard at its default time filter, after 10–15 minutes of traffic. Every panel should render data. A blank panel usually means: (1) the data view points at the wrong index, (2) the field name is wrong (often the case for cross-index panels), or (3) the time filter excludes everything because the runtime field's emit logic returned `null`.

---

## Translation Table — for Readers Still on Parts 5–8 Indexes

If you skipped Part 11's reshape and are still writing to `mule-logs` / `mule-traces` (OTLP-shaped, `Body` / `SeverityText` / `Resource.*` / per-event keys at top level), use this translation when copying the queries above:

| In the recipes above (ECS-shaped) | On the older OTLP-shaped index |
| --- | --- |
| `message` (logs) | `Body` |
| `log.level` (logs) | `SeverityText` |
| `service.name` (logs) | `Resource.service.name` |
| `service.environment` (logs) | `Resource.deployment.environment` |
| `service.node.name` (logs) | `Resource.workerId` |
| `host.id` (logs) | `Resource.workerId` |
| `process.thread.name` (logs) | `thread.name` |
| `trace.id` (logs) | `TraceId` (top-level) or `trace_id` (lowercase MDC bridge) |
| `labels.correlation_id` (logs) | `correlationId` (top-level, camelCase) |
| `mule.flow.processor_path` (logs) | `processorPath` (top-level) |

Trace-side field names did not change between Parts 7 and 11 — the recipes against `mule-traces-ecs` work unchanged on `mule-traces`.

---

## What We Covered

- A complete cheat sheet for the field-name asymmetry between the ECS-shaped log and trace indexes.
- **Twenty-five KQL recipes** in three tiers, all banking-flavored, all copy-pasteable.
- **Eight alert rules** ready to drop into Kibana Alerting, each with concrete thresholds, KQL, and grouping.
- **Four dashboards** — on-call, app-team, executive, audit/deploy — each with panel-by-panel Lens configurations.
- **Cross-index drilldowns** that turn one click into a complete request timeline.
- A **translation table** for readers still on the OTLP-shaped indexes from Parts 5–8.

The series is now complete: the cluster is up (Parts 1–3), the apps emit logs and traces (Parts 4–7), the data is reshaped and standardized (Parts 9–11), and we know how to query it, alert on it, and visualize it (this post). Next stop is the follow-up series on **Elastic APM Server**, which removes the OTel Collector and lets Kibana's Observability UI consume Mule traces natively — same Mule properties, fewer moving parts, deeper UI experience.

> ➡️ **Coming next — a follow-up series.** *"From OTel Collector to Elastic APM"* — swap the collector for APM Server, get service maps and flame graphs out of the box, and see what Kibana-native APM looks like over the same Mule 4.11 / Direct Telemetry Stream foundation we built here.

---

## References

- [Kibana Query Language (KQL) — Kibana Docs](https://www.elastic.co/guide/en/kibana/current/kuery-query.html)
- [Kibana Alerting — Rules and Connectors](https://www.elastic.co/guide/en/kibana/current/alerting-getting-started.html)
- [Lens — Kibana Docs](https://www.elastic.co/guide/en/kibana/current/lens.html)
- [Runtime fields — Elasticsearch Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/runtime.html)
- [Dashboard drilldowns — Kibana Docs](https://www.elastic.co/guide/en/kibana/current/drilldowns.html)
- [Elastic Common Schema — Field Reference](https://www.elastic.co/guide/en/ecs/current/ecs-field-reference.html)
- [Part 8 — Unified Kibana Dashboard](#)
- [Part 11 — Reshaping OTLP Logs into ECS-Native JSON](#)
