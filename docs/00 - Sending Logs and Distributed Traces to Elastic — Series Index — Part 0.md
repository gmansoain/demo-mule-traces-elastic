---
title: "Sending Logs and Distributed Traces to Elastic — Series Index — Part 0"
slug: "distributed-traces-with-elk-series-index"
description: "An eight-part series that builds an HTTP-only Elastic stack on AWS, two Mule 4.11 log-shipping paths, distributed tracing via OpenTelemetry, and a unified Kibana dashboard — read in any order, but designed to flow."
author: "Gonzalo Marcos"
date: 2026-06-09
status: not validated
lang: en
category: observability
tags:
  - elasticsearch
  - kibana
  - mule-runtime
  - logging
  - architecture-diagram
  - english
  - series
  - reference
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 0
type: series-index
difficulty: intermediate
read_time: 7
mule_version: "4.11"
platform:
  - aws
  - anypoint-platform
  - cloudhub-2
  - runtime-fabric
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![AWS](https://img.shields.io/badge/AWS-FF9900?logo=amazonaws&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Series_Index](https://img.shields.io/badge/Type-Series_Index-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 0](https://img.shields.io/badge/Part-0-16a085) ![7 min](https://img.shields.io/badge/Read_Time-7_min-lightgrey)

# Sending Logs and Distributed Traces to Elastic — Series Index — Part 0

A Mule app in production produces three things every operator wants to see together: **logs** (what the app said), **traces** (how the app's calls fanned out across services and how long each step took), and **metrics** (how it is behaving in aggregate). This series tackles the first two end-to-end on a self-managed Elastic stack — the kind we can stand up in an afternoon on AWS and own completely.

We will build an HTTP-only Elasticsearch and Kibana on EC2 (deliberately demo-grade, so the focus stays on the wiring), wire two complementary log-shipping paths from a Mule 4.11 app, add OpenTelemetry-based distributed tracing across two Mule services, and finish on a Kibana dashboard that joins logs and traces by `trace.id`. By the end we will be able to click a slow trace and see exactly which log lines accompanied each span.

This index page lays out what is in the series, who it is for, and the order to read it in. If you already know which post you want, jump to the [Series Index](#series-index) table and start there.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Start with:** [Part 1 — Installing Elasticsearch on Ubuntu 26.04 (HTTP-only) on AWS EC2](#)

> [!WARNING]
> **HTTP-only, demo-grade end to end.** Every post in this series puts the simpler path on the page first. Elasticsearch's HTTP layer runs without TLS, the OTel Collector listens on plain HTTP, and credentials cross the wire as Basic Auth headers. This is acceptable inside a private VPC with a tight security group. **Do not** copy this posture into a shared network or a real production deployment without re-reading the sister series, *Elastic Stack for MuleSoft Observability*, which covers the TLS-secured variant.

---

## What This Series Covers

- **Standing up Elasticsearch and Kibana on AWS** with `xpack.security` enabled but only HTTP TLS turned off, and least-privilege service users for the Mule apps to authenticate as.
- **Two log-shipping paths from Mule 4.11 to Elastic**, side by side:
  - A **Log4j2 HTTP appender** packaged inside the Mule app — works on every Mule target, no Anypoint subscription requirement.
  - **Direct Telemetry Stream** — Mule 4.11+'s built-in OTLP exporter, configured per app via Runtime Manager properties, no `log4j2.xml` customization, requires Advanced/Titanium tier.
- **A conceptual deep-dive** on what "OpenTelemetry support in Mule" actually means, why OTLP/protobuf cannot be POSTed straight to Elasticsearch's `/_doc` endpoint, and the seven realistic alternatives (Log4j2, Filebeat, Logstash, Elastic APM Server, OTel Collector, native OTLP, Anypoint Monitoring's built-in exporter).
- **Distributed tracing across two Mule apps** via OpenTelemetry, with W3C `traceparent` propagation between an upstream and downstream service. Trace context is also injected into the Log4j MDC so every `mule-logs` document carries the matching `trace.id`.
- **A unified Kibana dashboard** that joins `mule-logs` and `mule-traces` on `trace.id`, with a column-level drill-down from a slow-traces table straight to the matching log lines. Exported as portable NDJSON for one-click import on any future cluster.

---

## Who Is This For?

This series is aimed at MuleSoft architects, platform engineers, and senior developers who:

- Run (or are about to run) Mule apps on **CloudHub 2.0**, **Runtime Fabric**, or self-managed standalone, and need the logs and traces somewhere they can query.
- Are comfortable with `apt`, `systemctl`, EC2 security groups, and basic Mule project layout.
- Want to **own** the observability stack instead of leasing it from a vendor — but want a working demo first, before deciding which production posture to harden into.
- Are evaluating **OpenTelemetry vs. Log4j2 HTTP appender vs. Anypoint Monitoring** for their org and need a side-by-side feel for the trade-offs.

If you are looking for a TLS-secured, ECS-template-driven, ILM-managed production deployment, the sister series *Elastic Stack for MuleSoft Observability* is the better starting point — this one optimizes for clarity over hardening.

---

## Series Index

| Part | Title | Type | Difficulty |
| :---: | --- | --- | :---: |
| **1** | [Installing Elasticsearch on Ubuntu 26.04 (HTTP-only) on AWS EC2](#) | Tutorial | Intermediate |
| **2** | [Installing Kibana on Ubuntu 26.04 on a Separate AWS EC2 Instance](#) | Tutorial | Intermediate |
| **3** | [Initial Elasticsearch Config — Indexes, Roles, and Users for Mule Logs and Traces](#) | Tutorial | Intermediate |
| **4** | [Hello-World Mule App — Log4j2 HTTP Appender to Elasticsearch](#) | Tutorial | Intermediate |
| **5** | [Hello-World Mule App on CloudHub 2.0 and Runtime Fabric — Direct Telemetry Stream for Logs (No Log4j2)](#) | Tutorial | Advanced |
| **6** | [Why OpenTelemetry Cannot Ship Straight to Elastic — OTLP, Protobuf, and the Alternatives](#) | Deep-Dive | Intermediate |
| **7** | [Distributed Traces from Mule via OpenTelemetry to OTel Collector and Elasticsearch](#) | Tutorial | Advanced |
| **8** | [Unified Kibana Dashboard — Logs and Traces Across Mule Apps](#) | Tutorial | Intermediate |

The eight posts are designed to be read in order, but Parts 4 and 5 are deliberately interchangeable — pick the log-shipping path that matches your Mule version and Anypoint subscription tier, and read the other one later as a comparison. Part 6 reads cleanly on its own as a conceptual primer if you are not yet ready to deploy anything.

---

## Architecture We Will Build

```
                  ┌──────────────────────────────────────────────┐
                  │  Anypoint runtime (CH2 / RTF / Standalone)   │
                  │                                              │
                  │  ┌────────────┐         ┌────────────┐       │
                  │  │  App A     │  HTTP   │  App B     │       │
                  │  │  /hello    │────────▶│ /downstream│       │
                  │  └─────┬──────┘  +trace └─────┬──────┘       │
                  │        │ parent              │ child         │
                  │        ▼                     ▼               │
                  │   Direct Telemetry Stream (Mule 4.11+)       │
                  │   ─ logs exporter  ─ tracer exporter         │
                  └──────────────┬───────────────────────────────┘
                                 │ OTLP / HTTP (binary protobuf)
                                 ▼
                  ┌──────────────────────────────────────────────┐
                  │  EC2 — OpenTelemetry Collector (Contrib)     │
                  │   receivers: otlp                            │
                  │   exporters: elasticsearch/logs              │
                  │              elasticsearch/traces            │
                  └──────────────┬───────────────────────────────┘
                                 │ HTTP + Basic Auth
                                 ▼
       ┌──────────────────────────────────────────────────────────┐
       │  AWS — private VPC                                       │
       │                                                          │
       │   ┌─────────────────────┐         ┌─────────────────┐    │
       │   │ EC2 — Elasticsearch │◀────────│ EC2 — Kibana    │    │
       │   │  10.0.1.20:9200     │   HTTP  │  10.0.1.21:5601 │    │
       │   │  (HTTP-only,        │         │  (kibana_system │    │
       │   │   Basic Auth)       │         │   service user) │    │
       │   │                     │         └─────────────────┘    │
       │   │  mule-logs          │                                │
       │   │   ↑ mule-logger     │                                │
       │   │  mule-traces        │                                │
       │   │   ↑ mule-tracer     │                                │
       │   └─────────────────────┘                                │
       └──────────────────────────────────────────────────────────┘
                                 ▲
                                 │ Browser
                                 ▼
                ┌──────────────────────────────────────────┐
                │  Kibana — "Mule Observability" dashboard │
                │   ─ log volume by service                │
                │   ─ error rate per service               │
                │   ─ span p95 by service                  │
                │   ─ slow traces → click → log lines      │
                └──────────────────────────────────────────┘
```

Optional second path (Part 4) — the Log4j2 HTTP appender bypasses the OTel Collector and POSTs JSON straight from the app to `mule-logs`. Parts 4 and 5 can coexist on the same cluster; Parts 4, 6, and 7's troubleshooting tables explain how to keep the two writers from colliding on the index mapping.

---

## Prerequisites

To follow the whole series end-to-end:

- An **AWS account** with permissions to launch EC2 instances, edit security groups, and reach the VPC from a developer machine.
- An **Anypoint Platform** organization. The full series benefits from an **Advanced** or **Titanium** tier (Direct Telemetry Stream gates on it); Parts 1–4 work on any tier.
- A development machine with **Anypoint Studio 7.x** or **Anypoint Code Builder** targeting **Mule 4.11**, and **JDK 17**.
- Basic comfort with `apt`, `systemd`, and SSH on Ubuntu Server.
- About a half-day of focused time to follow Parts 1–7 end-to-end. Part 8 is its own afternoon.

The exact prerequisite list per post lives at the top of each tutorial.

---

## Conventions Used Across the Series

- **HTTP-only callout.** Every post opens with the same `[!WARNING]` calling out the demo-grade posture so a copy-paste into production cannot happen by accident.
- **Field names.** When OTel Collector field shapes vary by version (e.g. `Attributes.trace.id` on logs vs `TraceId` on traces), the post's prerequisites table is the source of truth for that post.
- **Property casing.** MuleSoft's OpenTelemetry properties use camel-case `T` — `mule.openTelemetry.tracer.exporter.enabled`, *not* `mule.opentelemetry.*` and *not* `mule.otel.*`. The wrong casing fails silently. Every post calls this out where it applies.
- **Verbatim values.** Property names, endpoint paths, and CLI flags are copied directly from the upstream docs. Where a value comes from the MuleSoft / OTel / Elastic documentation, it is linked in the **References** section of that post.
- **Repo assets.** Mule app sources, OTel Collector configs, Log4j2 layouts, and the Kibana dashboard NDJSON live under `./assets/<topic>/` in the same folder as the posts. Each tutorial links to the exact files it uses.

---

## Where Each Post Picks Up From the Previous One

If we miss the explicit dependencies between posts they look like a flat list — they are not. Here is the actual graph:

- **Part 1 → Part 2.** Part 2's Kibana host needs the private IP of the Elasticsearch host and the `kibana_system` password reset on it.
- **Part 2 → Part 3.** Part 3 uses the running Kibana (and the `elastic` superuser) to create indexes, roles, and service users.
- **Part 3 → Part 4 / Part 5.** Both Mule paths consume the `mule-logger` user and the `mule-logs` index from Part 3. They can be done in either order — pick the one that matches your subscription tier.
- **Part 5 → Part 6 → Part 7.** Part 6 is a conceptual interlude that explains the *why* behind Part 5's OTel Collector. Part 7 reuses the same collector for traces and assumes the systemd drop-in pattern for credentials from Part 5 Step 3.
- **Part 7 → Part 8.** Part 8's dashboard reads from both `mule-logs` (populated by Part 5 and/or Part 4) and `mule-traces` (populated by Part 7).

If you skip a post, the subsequent post's prerequisites section calls out exactly what is missing.

---

## What This Series Deliberately Does *Not* Cover

So that we do not pretend to be production:

- **TLS / mTLS.** Sister series *Elastic Stack for MuleSoft Observability* covers HTTP TLS replacement, the JVM truststore, app-bundled JKS strategies, and certificate rotation runbooks.
- **Index Lifecycle Management (ILM).** Real production indexes need rollover, shard count tuning, and retention policies. Out of scope here; the same sister series covers it for `mule-logs`.
- **Elastic APM Server.** A natural successor — APM Server speaks OTLP natively, writes to APM data streams, and unlocks Kibana's Observability UI (service map, latency distributions, span flame graphs). Will get its own series.
- **Metrics ingestion.** Mule 4.12 added Direct Telemetry Stream for metrics. We will get to it after APM.
- **Alerting and SLOs.** Once dashboards exist, the next layer is Watcher / Alerting on top of `mule-logs` and `mule-traces`. Future posts.
- **Filebeat / Fluent Bit / Logstash.** Discussed in Part 6 as alternatives, but no tutorial in this series. Filebeat as a sidecar on RTF is the most likely standalone follow-up.

---

## Recommended Reading Order

If you have **a half-day** and a fresh AWS account: read **Parts 1 → 2 → 3 → 4 → 8**. That gets a single Mule app shipping logs to a working dashboard.

If you have **a full day** and want the whole series payoff: read **Parts 1 → 2 → 3 → 5 → 6 → 7 → 8** (skip Part 4 unless you are also evaluating the Log4j2 path).

If you are **just choosing an approach** and have not deployed anything yet: read **Part 6** first. The trade-off table at the end will tell you whether the rest of the series is worth your time before you commit to any infrastructure.

If you are **debugging a problem** in an existing setup: each post's troubleshooting table is self-contained. Search this folder for the symptom string from your error message — the series tries to encode the failure modes it has actually hit during writing.

---

## What Comes After the Series

- **From OTel Collector to Elastic APM** — swap the OTel Collector + plain `mule-traces` index for **Elastic APM Server**, which natively ingests OTLP and writes to APM data streams. Same Mule properties, one fewer component to operate, deeper Kibana-native experience (service map, latency distributions, error groupings).
- **Filebeat as a sidecar on RTF** — for orgs without Advanced/Titanium tier, the cleanest no-OTel path on Runtime Fabric.
- **Hardening the demo** — TLS-terminating proxy in front of the HTTP-only cluster, ILM policies, ECS-native templates, alerting on the dashboards we built in Part 8.

---

## References

- [OpenTelemetry support for Mule runtime — MuleSoft Docs](https://docs.mulesoft.com/mule-runtime/latest/otel-support)
- [OTLP Specification — OpenTelemetry](https://opentelemetry.io/docs/specs/otlp/)
- [Anypoint Connector for HTTP — Release Notes](https://docs.mulesoft.com/release-notes/connector/connector-http)
- [W3C Trace Context — `traceparent` Specification](https://www.w3.org/TR/trace-context/)
- [Elasticsearch Exporter — OTel Collector Contrib](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/elasticsearchexporter)
- [Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/index.html)
- [Mule 4.11 Release Notes — MuleSoft Docs](https://docs.mulesoft.com/release-notes/mule-runtime/mule-4.11.0-release-notes)
- [Lens — Kibana Docs](https://www.elastic.co/guide/en/kibana/current/lens.html)
- [Sister series — *Elastic Stack for MuleSoft Observability*](#)
