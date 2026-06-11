---
title: "Hello-World Mule App on CloudHub 2.0 and Runtime Fabric — Direct Telemetry Stream for Logs (No Log4j2) — Part 5"
slug: "mule-direct-stream-cloudhub-rtf-elasticsearch-logs"
description: "Deploy the same Mule 4.11 hello-world app to CloudHub 2.0 and Runtime Fabric and stream logs straight from the runtime as OTLP — no custom log4j2.xml — through an OpenTelemetry Collector into Elasticsearch."
author: "Gonzalo Marcos"
date: 2026-06-08
status: not validated
lang: en
category: observability
tags:
  - mule-runtime
  - cloudhub-2
  - runtime-fabric
  - logging
  - anypoint-platform
  - english
  - series
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 5
type: tutorial
difficulty: advanced
read_time: 16
mule_version: "4.11"
platform:
  - cloudhub-2
  - runtime-fabric
  - anypoint-platform
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![CloudHub 2.0](https://img.shields.io/badge/Platform-CloudHub_2.0-00A0DF?logo=mulesoft&logoColor=white) ![Runtime Fabric](https://img.shields.io/badge/Platform-Runtime_Fabric-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Advanced](https://img.shields.io/badge/Level-Advanced-e74c3c) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 5](https://img.shields.io/badge/Part-5-16a085) ![16 min](https://img.shields.io/badge/Read_Time-16_min-lightgrey)

# Hello-World Mule App on CloudHub 2.0 and Runtime Fabric — Direct Telemetry Stream for Logs (No Log4j2) — Part 5

In [Part 4](#) we wired a hello-world Mule 4.11 app to Elasticsearch through Log4j2's `HttpAppender`. That works everywhere a JVM runs, but it ties log shipping to the **app's** packaging — every project carries its own `log4j2.xml`, its own credentials wiring, and its own appender to maintain. From Mule **4.11.0** onward there is a different option: **Direct Telemetry Stream** — the runtime itself emits logs (and traces) as **OTLP** over HTTP or gRPC, configured entirely with Runtime Manager properties.

In this tutorial we will deploy the **same** hello-world app from Part 4 to CloudHub 2.0 and to Runtime Fabric, but the project ships **without a custom `log4j2.xml`**. Instead, we will turn on Direct Telemetry Stream's **logs exporter** with three Runtime Manager properties, point it at a tiny OpenTelemetry Collector, and have the collector's Elasticsearch exporter write into the `mule-logs` index from Part 3. The Mule app does not know anything is different.

This is intentionally a stepping-stone toward Part 6: the same OTel Collector that we set up here for **logs** will also receive **traces** in Part 6, so we get the collector deployment out of the way once.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 4 — Hello-World Mule App: Log4j2 HTTP Appender → Elasticsearch](#) · **Next:** Part 6 — Why OpenTelemetry Cannot Ship Straight to Elastic — OTLP, Protobuf, and the Alternatives

> [!WARNING]
> **HTTP-only, demo-grade.** The OTel Collector listens on plain HTTP, exports to Elasticsearch over plain HTTP, and the Mule runtime ships OTLP/HTTP without TLS. Acceptable inside a private VPC where the Anypoint VPC / RTF cluster can reach the collector and Elasticsearch over the AWS internal network; **not** appropriate for shared networks. Mule's Direct Telemetry Stream supports mTLS via `mule.openTelemetry.exporter.tls.*` for the production variant.

> [!IMPORTANT]
> **Direct Telemetry Stream requires:**
> - **Mule runtime 4.11.0** or later (we are using 4.11 — this is the minimum for the series).
> - An **Advanced** or **Titanium** subscription tier of MuleSoft Anypoint Platform.
>
> If our org is on the base subscription, the Part 4 Log4j2 path is the right call — same destination, just packaged in the app.
> Source: [MuleSoft docs — OpenTelemetry support](https://docs.mulesoft.com/mule-runtime/latest/otel-support).

---

## What We Will Cover

- Reuse the hello-world Mule 4.11 app from Part 4, **stripped of its custom `log4j2.xml`**.
- Stand up a small **OpenTelemetry Collector** on a third EC2 instance, with an `otlp` receiver and an `elasticsearch` exporter that writes to the `mule-logs` index.
- Enable **Direct Telemetry Stream — Logs Exporter** on the Mule runtime with three Runtime Manager properties.
- Deploy the app twice — once to CloudHub 2.0 and once to Runtime Fabric — and confirm both runtimes stream OTLP logs to the collector and land in `mule-logs`.
- Compare Direct Telemetry Stream against the Log4j2 path so we can pick deliberately for future apps.

---

## Prerequisites

Before we start, we will need:

- The hello-world Mule project from [Part 4](#) — same flow, same `pom.xml`, on **Mule 4.11**.
- A working Elasticsearch + Kibana from [Parts 1–3](#), reachable from the OTel Collector host.
- The `mule-logger` user credentials from [Part 3](#) and the base64 `Authorization` value (from Part 4 Step 3).
- A third Ubuntu Server 26.04 LTS EC2 instance for the OTel Collector. We will call its private IP `10.0.1.30`.
- Network reachability:
  - Mule runtime → OTel Collector on `4318/tcp` (OTLP/HTTP). Mule's Direct Telemetry Stream defaults to gRPC on `4317`, but we will use HTTP for parity with the rest of the series.
  - OTel Collector → Elasticsearch on `9200/tcp`.
- An **Anypoint Platform** organization with **Advanced** or **Titanium** tier in the environment we will deploy to.
- A **CloudHub 2.0 private space** and a **Runtime Fabric** environment registered in Anypoint.

We will sanity-check the cluster is reachable from the would-be collector host before deploying:

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_count?pretty"
```

We expect a JSON response with the running count of documents from Part 3 + Part 4.

---

## Why Direct Telemetry Stream Instead of `log4j2.xml`?

Both paths land documents in the same `mule-logs` index. The difference is *where* the configuration lives, *who* owns it, and *what* it costs to add the next app.

| Approach | Pros | Cons |
| --- | --- | --- |
| **Log4j2 HTTP appender (Part 4)** | Works on every Mule target — Studio, ACB, standalone, CH1/2, RTF. Full control of the JSON shape and the layout. Survives an OTel Collector outage because the appender talks straight to Elastic. No paid Anypoint tier required. | Each app carries its own `log4j2.xml`, its own credentials wiring, and its own appender to maintain. Truststore work in the HTTPS variant. Rotating the destination or the credential is a redeploy of every app. |
| **Direct Telemetry Stream (this post)** | Configured **per-app via Runtime Manager properties** — no app code, no `log4j2.xml`, no truststore. Standard **OTLP** wire format — same collector serves logs *and* traces (Part 6). Rotating destination/credentials is a properties change + restart. | Requires Mule **4.11.0+** and **Advanced/Titanium** subscription. Anypoint Monitoring's OpenTelemetry features (Telemetry Exporter) become unavailable while this is on. Coupled to OTLP/Collector — the demo's Mule logs go nowhere if the collector is down. |

| Decision | Options Considered | Chosen | Rationale |
| --- | --- | --- | --- |
| Wire format from Mule | Log4j2 HTTP-JSON / OTLP-HTTP / OTLP-gRPC | OTLP-HTTP (`/v1/logs`) | OTLP is the standard; HTTP keeps debugging easy with `curl`; the same collector receives Part 6's traces |
| Where to define exporter | per-app Runtime Manager properties / per-environment | per-app | The docs document only the per-app properties path; matches CH2/RTF UI flows |
| Backend path | Mule → Elastic direct OTLP / Mule → Collector → Elastic | Mule → Collector → Elastic | Mule's docs explicitly recommend going through a collector for vendors that cannot ingest OTLP natively, including older Elastic versions |

---

## Overview

```
┌────────────────────────────┐         ┌────────────────────────────┐         ┌────────────────────────────┐
│  Anypoint runtime          │         │  AWS — private VPC         │         │  AWS — private VPC         │
│  (CH2 / RTF)               │ OTLP    │  EC2 — OTel Collector      │ HTTP    │  EC2 — Elasticsearch       │
│                            │  HTTP   │   10.0.1.30:4318           │         │   10.0.1.20:9200           │
│  hello-world-mule          │────────▶│   receivers: otlp          │────────▶│   index: mule-logs         │
│  (no log4j2.xml)           │         │   exporters: elasticsearch │         │                            │
│                            │         └────────────────────────────┘         └────────────────────────────┘
│  Direct Telemetry Stream   │
│  Logs Exporter ON          │
└────────────────────────────┘
```

---

## Table of Contents

1. [Step 1 — Strip the Custom `log4j2.xml` from the App](#step-1--strip-the-custom-log4j2xml-from-the-app)
2. [Step 2 — Provision the OTel Collector EC2 Instance](#step-2--provision-the-otel-collector-ec2-instance)
3. [Step 3 — Configure the OTel Collector for Logs](#step-3--configure-the-otel-collector-for-logs)
4. [Step 4 — Build and Publish the Artifact to Exchange](#step-4--build-and-publish-the-artifact-to-exchange)
5. [Step 5 — Deploy to CloudHub 2.0 with Direct Telemetry Stream](#step-5--deploy-to-cloudhub-20-with-direct-telemetry-stream)
6. [Step 6 — Deploy to Runtime Fabric with Direct Telemetry Stream](#step-6--deploy-to-runtime-fabric-with-direct-telemetry-stream)
7. [Step 7 — Trigger Both Apps and Verify in Kibana](#step-7--trigger-both-apps-and-verify-in-kibana)
8. [Verification](#verification)
9. [Troubleshooting](#troubleshooting)

---

### Step 1 — Remove the HTTP Appender from `log4j2.xml`

We will start from the Part 4 project. The Mule flow stays exactly the same — we are *not* changing `src/main/mule/hello-world.xml`. The change is **inside `log4j2.xml`**: we drop the `<Http>` appender (which sent each event as JSON to Elastic over HTTP) and keep everything else.

> [!IMPORTANT]
> **"Strip" does not mean delete the file.** Earlier drafts of this post said `rm src/main/resources/log4j2.xml`, which works only when the project has no other Log4j customization worth keeping. In practice, every real project carries logger-level overrides (`org.mule.service.http` at WARN, custom levels for noisy packages, etc.) that we want to preserve. Keep `log4j2.xml`; just remove the `<Http>` appender block and its `<AppenderRef>`.

The post-edit file should look like this — `<RollingFile>` only, no `<Http>` appender, no `<AppenderRef ref="Elastic"/>`:

📄 `src/main/resources/log4j2.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Configuration>

    <Appenders>
        <RollingFile name="file"
                     fileName="${sys:mule.home}${sys:file.separator}logs${sys:file.separator}hello-world-mule.log"
                     filePattern="${sys:mule.home}${sys:file.separator}logs${sys:file.separator}hello-world-mule-%i.log">
            <JsonLayout properties="true"
                        compact="true"
                        eventEol="true"
                        stacktraceAsString="true"
                        includeTimeMillis="true">
                <KeyValuePair key="@timestamp"
                              value="$${date:yyyy-MM-dd'T'HH:mm:ss.SSSXXX}"/>
                <KeyValuePair key="service.name" value="hello-world-mule"/>
            </JsonLayout>
            <SizeBasedTriggeringPolicy size="10 MB"/>
            <DefaultRolloverStrategy max="10"/>
        </RollingFile>
    </Appenders>

    <Loggers>
        <!-- Trim noisy packages so on-disk and Anypoint Logs view stay readable. -->
        <AsyncLogger name="org.mule.service.http" level="WARN"/>
        <AsyncLogger name="org.mule.extension.http" level="WARN"/>

        <!-- Keep Mule's Logger processor at INFO so the OTel exporter sees the events. -->
        <AsyncLogger name="org.mule.runtime.core.internal.processor.LoggerMessageProcessor"
                     level="INFO"/>

        <AsyncRoot level="INFO">
            <AppenderRef ref="file"/>
        </AsyncRoot>
    </Loggers>

</Configuration>
```

📄 Full file: [./assets/hello-world-mule-direct-stream/src/main/resources/log4j2.xml](./assets/hello-world-mule-direct-stream/src/main/resources/log4j2.xml)

> [!NOTE]
> **How Direct Telemetry Stream finds these events.** The Mule docs are explicit: the OTel logs exporter taps into the **Log4j event stream in-process** — it does not read the on-disk file. That means two things: (1) the file appender above is for human consumption (live tail in Runtime Manager / RTF logs view), not for Elastic; (2) the `<AsyncRoot level="INFO">` and the `LoggerMessageProcessor` logger level are what gate whether the OTel exporter sees the event in the first place. Lower the root to WARN and OTel ships nothing — `mule.openTelemetry.logging.exporter.level=INFO` is a *filter on the exporter side*, not a request to lift Log4j's root level.

> [!WARNING]
> **Do not run Part 4's HTTP appender and Part 5's OTel Direct Stream against the *same* `mule-logs` index.** They emit different field shapes — `JsonLayout`'s flat `thread` string vs OTLP's `thread.id` / `thread.name` object — and Elasticsearch's dynamic mapping locks on whichever wrote first. The other writer's documents are then rejected with `document_parsing_exception`. Keep the HTTP appender out of this project (Step 1) **and** make sure no Part 4 instance is still pointing at `mule-logs`. If both writers must coexist on the same cluster, see the recommendation at the end of Part 4's troubleshooting table — separate indexes per writer (`mule-logs-log4j` vs `mule-logs-otlp`) or one index with `mapping.mode: ecs` on the OTel exporter.

We will also rename the artifact so the two deployments are distinguishable from the Anypoint UI:

📄 `pom.xml`

```xml
<artifactId>hello-world-mule-direct-stream</artifactId>
<version>1.0.0</version>
```

---

### Step 2 — Provision the OTel Collector EC2 Instance

We will launch a third EC2 instance in the same VPC as the Elasticsearch host so the cross-instance traffic stays on the AWS internal network.

- AMI: **Ubuntu Server 26.04 LTS**.
- Instance type: `t3.small` (2 vCPU / 2 GB) is enough for this demo's volume.
- Storage: 10 GB gp3.
- Security group `otel-collector-sg`:
  - Inbound `22/tcp` from our admin IP.
  - Inbound `4318/tcp` from the **CloudHub 2.0 private space NAT** and the **RTF cluster CIDR** (the OTLP/HTTP receiver port).
  - Inbound `4317/tcp` if we ever want to switch to gRPC.

We will then update the **Elasticsearch SG** to allow inbound `9200/tcp` from `otel-collector-sg`.

After the instance is up, we will rename it (same reasons as Parts 1 and 2):

```bash
sudo hostnamectl set-hostname mule-elk-otelcol-01
echo "127.0.1.1 mule-elk-otelcol-01" | sudo tee -a /etc/hosts
sudo sed -i 's/^preserve_hostname:.*$/preserve_hostname: true/' /etc/cloud/cloud.cfg
```

We will install the OpenTelemetry Collector Contrib distribution (the `elasticsearch` exporter ships in `contrib`, not in `core`):

```bash
OTELCOL_VERSION="0.110.0"
wget -q "https://github.com/open-telemetry/opentelemetry-collector-releases/releases/download/v${OTELCOL_VERSION}/otelcol-contrib_${OTELCOL_VERSION}_linux_amd64.deb"
sudo dpkg -i "otelcol-contrib_${OTELCOL_VERSION}_linux_amd64.deb"
```

![](Hello-World%20Mule%20App%20on%20CloudHub%202.0%20and%20Runtime%20Fabric%20—%20Direct%20Stream%20Logging%20(No%20Log4j2)%20—%20Part%205.png)

The package installs `/usr/bin/otelcol-contrib`, a default config at `/etc/otelcol-contrib/config.yaml`, and a systemd unit `otelcol-contrib.service`.



---

### Step 3 — Configure the OTel Collector for Logs

We will replace the default config with one that:

- accepts OTLP over HTTP on `:4318` (logs and traces — traces are unused in this post but ready for Part 6),
- exports logs to Elasticsearch's `mule-logs` index using the `mule-logger` Basic Auth credentials.

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
  elasticsearch:
    endpoints: ["http://10.0.1.20:9200"]
    user: mule-logger
    password: ${env:MULE_LOGGER_PASSWORD}
    logs_index: mule-logs
    # Bypass dynamic mapping inference; we are writing to a plain index.
    mapping:
      mode: raw

service:
  pipelines:
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [elasticsearch]
```

📄 Full file: [./assets/otel-collector/config.yaml](./assets/otel-collector/config.yaml)

#### 3.1 — Inject the `mule-logger` password via a systemd drop-in

The collector resolves `${env:MULE_LOGGER_PASSWORD}` at startup from its **process environment** — and a process started by systemd inherits *only* what systemd hands it. A shell `export MULE_LOGGER_PASSWORD=...` in our SSH session is invisible to the daemon. We will tell systemd to inject the variable.

We will use a **drop-in override** rather than editing the package-shipped unit file directly. Two reasons: a `dpkg` upgrade will not clobber a drop-in, and the resulting file is small and self-contained, easy to audit later.

Open the editor:

```bash
sudo systemctl edit otelcol-contrib
```

> [!IMPORTANT]
> `systemctl edit` (no flags) is what we want. It opens an empty editor scoped to a drop-in file at `/etc/systemd/system/otelcol-contrib.service.d/override.conf`. **Do NOT** run `systemctl edit --full` — that flag opens the package-shipped unit, and any changes there are lost on the next `dpkg` upgrade. Always prefer the drop-in.

In the empty editor, paste **both lines below — including the `[Service]` header** at the top:

```ini
[Service]
Environment="MULE_LOGGER_PASSWORD=<MULE_LOGGER_PASSWORD>"
```

Save and close. The default editor is `nano` on Ubuntu (`Ctrl+O`, `Enter`, `Ctrl+X`); set `SYSTEMD_EDITOR=vim` if you prefer `vim`.

> [!WARNING]
> The `[Service]` header is **mandatory** — without it, systemd silently ignores the `Environment=` line because it does not know which section the directive belongs to. The most common reason for "the password is not picked up" is pasting only the second line.

#### 3.2 — Confirm the drop-in is on disk

```bash
sudo cat /etc/systemd/system/otelcol-contrib.service.d/override.conf
```

We expect to see **exactly** the two lines we pasted, with the `[Service]` header on the first line. If the file is empty or missing the header, re-run `sudo systemctl edit otelcol-contrib`.

#### 3.3 — Reload systemd, then start the unit

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now otelcol-contrib
sudo systemctl status otelcol-contrib
```

> [!IMPORTANT]
> **Why `daemon-reload` is required.** systemd parses every unit file (and every drop-in) once at boot and caches the result in memory. A new file under `/etc/systemd/system/.../override.conf` is just a text file on disk until we tell systemd to re-read its catalogue. `daemon-reload` is what triggers that re-read. Without it, `systemctl start` launches the collector with the **old** unit definition — which has no `Environment=MULE_LOGGER_PASSWORD=...` — and the collector ends up sending an empty password to Elasticsearch, which Elastic returns as `401`. Get the order right: **edit → daemon-reload → start/restart**, every time.
>
> If we are *changing* a drop-in (not creating one for the first time), the collector also needs to be **restarted**, not just reloaded — a running process keeps its environment from the moment it was forked. `sudo systemctl restart otelcol-contrib` is the second half of the recipe.

#### 3.4 — Verify the variable actually reached the running process

Two quick sanity checks before we move on. Both have caught real misconfigurations.

**A — Did systemd parse the drop-in?**

```bash
systemctl show otelcol-contrib -p Environment
```

We expect:

```text
Environment=MULE_LOGGER_PASSWORD=<our-value>
```

If we see an empty `Environment=` here, the drop-in did not parse — usually the `[Service]` header is missing.

**B — Is the variable in the live process's environment?**

```bash
PID=$(pgrep -f otelcol-contrib | head -1)
sudo tr '\0' '\n' < /proc/$PID/environ | grep MULE_LOGGER
```

We expect a `MULE_LOGGER_PASSWORD=...` line. If it is there but Elastic still rejects auth, the bug is in the collector's `config.yaml` (e.g. `${MULE_LOGGER_PASSWORD}` instead of `${env:MULE_LOGGER_PASSWORD}` — the OTel Collector requires the `env:` prefix from `v0.86` onward). If it is **not** there, we missed `daemon-reload` or `restart` in 3.3.

We will tail the journal until we see the `Everything is ready. Begin running and processing data.` line:

```bash
sudo journalctl -u otelcol-contrib -f
```

> [!TIP]
> Quick smoke test from the collector host — send a synthetic OTLP/HTTP log envelope and confirm a doc lands in `mule-logs`:
>
> ```bash
> curl -X POST http://localhost:4318/v1/logs \
>   -H 'Content-Type: application/json' \
>   -d '{"resourceLogs":[{"scopeLogs":[{"logRecords":[{"timeUnixNano":"'"$(date +%s)"'000000000","severityText":"INFO","body":{"stringValue":"otel collector smoke test"}}]}]}]}'
> ```
>
> Then in Kibana → Discover → **Mule Logs**, filter for `body: "otel collector smoke test"`. If we see it, the collector → Elastic leg works.


```
curl -X POST http://localhost:4318/v1/logs \
  -H 'Content-Type: application/json' \
  -d '{"resourceLogs":[{"scopeLogs":[{"logRecords":[{"timeUnixNano":"'"$(date +%s)"'000000000","severityText":"INFO","body":{"stringValue":"otel collector smoke test from Gon!"}}]}]}]}'
```


---

### Step 4 — Build and Publish the Artifact to Exchange

CH2 and RTF deploy from Anypoint Exchange. We will publish the JAR there once, then point both deployments at it.

```bash
mvn clean deploy -DskipTests
```

The Mule Maven Plugin's `deploy` goal pushes the artifact to Exchange. We expect a `BUILD SUCCESS` and the asset to show up under **Anypoint Exchange → Provided by &lt;our org&gt;**.

> [!NOTE]
> *Screenshot — Anypoint Exchange listing the `hello-world-mule-direct-stream` asset, version `1.0.0`.*

---

### Step 5 — Deploy to CloudHub 2.0 with Direct Telemetry Stream

We will deploy the artifact from Exchange and turn on the logs exporter via Runtime Manager properties.

1. **Anypoint Runtime Manager → Applications → Deploy Application**.
2. **Application Name:** `hello-world-mule-direct-stream-ch2`.
3. **Deployment target:** `CloudHub 2.0 → <our private space>`.
4. **Application file:** the asset from Step 4.
5. **Runtime version:** `4.11.0` (minimum for Direct Telemetry Stream).
6. **Replicas / vCores:** 1 replica, 0.1 vCores.
7. **Properties tab — add the following.** These are the **exact** property names from the MuleSoft docs:

| Property                                            | Value                                | Type     |
| --------------------------------------------------- | ------------------------------------ | -------- |
| `mule.openTelemetry.logging.exporter.enabled`       | `true`                               | Property |
| `mule.openTelemetry.logging.exporter.type`          | `HTTP`                               | Property |
| `mule.openTelemetry.logging.exporter.endpoint`      | `http://10.0.1.30:4318/v1/logs`      | Property |
| `mule.openTelemetry.logging.exporter.level`         | `INFO`                               | Property |
| `mule.openTelemetry.exporter.resource.service.name` | `hello-world-mule-direct-stream-ch2` | Property |
| `mule.put.trace.id.and.span.id.in.mdc`              | `true`                               | Property |

8. Click **Deploy Application**.

Wait for status `Started`.

> [!IMPORTANT]
> The properties live under the **`mule.openTelemetry.logging.exporter.*`** prefix, not `mule.opentelemetry.*` (no camel-case `T`) and not `mule.otel.*`. Get the casing wrong and the runtime silently falls back to the default (exporter disabled), which looks identical to a misconfigured collector.

> [!TIP]
> `mule.put.trace.id.and.span.id.in.mdc=true` injects the active `trace.id` and `span.id` into the Log4j MDC, so when traces start flowing in Part 6 the log records carry the same `trace.id` — that is what unlocks log↔trace correlation in Part 7's dashboard. The MuleSoft docs say this will become automatic in a future release; explicit is fine for 4.11.

> [!WARNING]
> "When using this direct telemetry stream feature, Anypoint Monitoring OpenTelemetry features, such as Telemetry Exporter, aren't available." (MuleSoft docs.) That is fine for our standalone Elastic stack, but if our org also uses Anypoint's native OTel pipeline we will be turning that off for this app.

> [!NOTE]
> *Screenshot — Runtime Manager application page on the **Properties** tab with the six rows above filled in, and the deployment in `Started` status.*

---

### Step 6 — Deploy to Runtime Fabric with Direct Telemetry Stream

The same artifact, same Mule version, deployed to RTF — and the exact same Properties.

1. **Anypoint Runtime Manager → Applications → Deploy Application**.
2. **Application Name:** `hello-world-mule-direct-stream-rtf`.
3. **Deployment target:** `Runtime Fabric → <our RTF cluster>`.
4. **Application file:** the same Exchange asset.
5. **Runtime version:** `4.11.0`.
6. **Replicas / Resources:** 1 replica, default resources.
7. **Properties tab — same six properties as CH2** (Step 5), but bump `mule.openTelemetry.exporter.resource.service.name` to `hello-world-mule-direct-stream-rtf` so the two deployments are distinguishable in the data.
8. Click **Deploy Application**.

> [!IMPORTANT]
> Direct Telemetry Stream property names are identical across CH2 and RTF (and on-prem standalone — although on standalone, `service.name` per app is not supported per the docs). The platform-specific differences live elsewhere — networking from the runtime to the collector, mTLS strategies — not in the property names.

> [!NOTE]
> *Screenshot — Runtime Manager application page for the RTF deployment, Properties tab with the same exporter properties.*

---

### Step 7 — Trigger Both Apps and Verify in Kibana

We will hit the `/hello` endpoint on each deployment a few times so we have something to look at:

```bash
# CH2
for i in 1 2 3; do curl -s https://hello-world-mule-direct-stream-ch2.<region>.cloudhub.io/hello; done

# RTF
for i in 1 2 3; do curl -s https://hello-world-mule-direct-stream-rtf.<rtf-ingress>/hello; done
```

Each call should return:

```json
{"message":"Hello World"}
```

Now switch to Kibana → **Analytics → Discover** → select the **Mule Logs** data view from Part 3.

We expect to see fresh documents from both deployments. The shape comes from the OTel Collector's Elasticsearch exporter (`mapping.mode: raw`), so the field names are OTLP/ECS-flavored — different from Part 4's `JsonLayout`. Common fields:

| Field | Example value |
| --- | --- |
| `@timestamp` | `2026-06-08T11:42:01.123Z` (from the OTLP `timeUnixNano`) |
| `Severity` (or `severity_text`) | `INFO` |
| `Body` (or `body`) | `Hello World - request received` |
| `Resource.service.name` | `hello-world-mule-direct-stream-ch2` |
| `Attributes.trace.id` | populated **only** when traces are also enabled (Part 6) |
| `Attributes.span.id` | same — populated only with traces on |

Filter by `Resource.service.name: hello-world-mule-direct-stream-*` to see only this post's deployments and confirm both targets are flowing.

> [!NOTE]
> *Screenshot — Kibana Discover with the **Mule Logs** data view, filtered by `Resource.service.name: hello-world-mule-direct-stream-*`, showing documents from both `-ch2` and `-rtf`.*

> [!TIP]
> If the time filter in Kibana hides the new documents and the data view's time field is `@timestamp` but the OTel Elasticsearch exporter writes to `Timestamp` (capital `T`), edit the data view in **Stack Management → Data Views → Mule Logs** and switch the time field to whatever the exporter emits. This depends on the exporter version — check one document's fields in Discover and pick the one that matches.

---

## Verification

We will run three checks to confirm both deployments are wired end-to-end.

**1. Both apps are running.**

In Anypoint Runtime Manager, both `hello-world-mule-direct-stream-ch2` and `-rtf` should show status `Started`. Each app's on-platform **Logs** tab should still show the `Hello World - request received` line — that comes from the runtime's default Log4j output, untouched by the OTel exporter.

**2. The OTel Collector is receiving data.**

```bash
# On the collector host
sudo journalctl -u otelcol-contrib -f | grep -i "logs"
```

We expect to see periodic `LogsExporter` activity. The Collector's default Prometheus metrics endpoint also exposes `otelcol_exporter_sent_log_records_total` and `otelcol_exporter_send_failed_log_records_total` on `:8888/metrics` — useful when troubleshooting.

**3. Elasticsearch shows the same documents.**

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_search?q=Body:%22Hello%20World%20-%20request%20received%22&size=20&pretty"
```

We expect at least six hits (three per deployment). Each should carry the `Resource.service.name` so we can tell them apart.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Properties saved but no documents in `mule-logs`, no app errors | Property name typo — must be exactly `mule.openTelemetry.logging.exporter.*` (camel-case `T` in `Telemetry`) | Re-check casing of every property; restart the app after correcting |
| Mule app log shows `Failed to export log records: io.opentelemetry...UNAVAILABLE` | Collector unreachable — wrong endpoint, blocked by SG, or collector down | `curl -v http://10.0.1.30:4318/v1/logs` from the runtime side; open `4318/tcp` from CH2 NAT / RTF cluster CIDR; `systemctl status otelcol-contrib` |
| Collector log shows `connection refused` to Elasticsearch | ES SG does not allow inbound from `otel-collector-sg`, or ES bound to `127.0.0.1` only | Add the SG rule on the ES instance; revisit Part 1 Step 6 |
| Collector log shows `401 Unauthorized` from Elasticsearch | The `mule-logger` password is empty or wrong from the collector's point of view — drop-in not picked up, missing `[Service]` header, missed `daemon-reload`/`restart`, or the collector resolves a literal `${MULE_LOGGER_PASSWORD}` because `config.yaml` is missing the `env:` prefix | Walk through the four checks in order: (1) `sudo cat /etc/systemd/system/otelcol-contrib.service.d/override.conf` — the file must contain both `[Service]` and `Environment="MULE_LOGGER_PASSWORD=..."`; (2) `systemctl show otelcol-contrib -p Environment` — must show the variable; if empty, the `[Service]` header is missing; (3) `PID=$(pgrep -f otelcol-contrib \| head -1); sudo tr '\0' '\n' < /proc/$PID/environ \| grep MULE_LOGGER` — must show the variable in the live process; if not, run `sudo systemctl daemon-reload && sudo systemctl restart otelcol-contrib`; (4) confirm `config.yaml` reads `password: ${env:MULE_LOGGER_PASSWORD}` (with `env:` prefix — required from OTel Collector v0.86 onward) |
| `Environment=` is empty in `systemctl show` | `[Service]` header missing from the drop-in | Re-run `sudo systemctl edit otelcol-contrib` and paste **both** `[Service]` and `Environment="..."` lines |
| Drop-in saved correctly but `/proc/$PID/environ` does not show the variable | systemd cache still holds the old unit definition, or the running process was forked before the change | `sudo systemctl daemon-reload` (re-reads the catalogue), then `sudo systemctl restart otelcol-contrib` (re-forks the process so it inherits the new env). `daemon-reload` alone is not enough |
| Password contains `$`, `\`, `!`, or `%` and auth still fails | systemd's `Environment=` parser strips or expands those characters | Reset the `mule-logger` password (Part 3 Step 3) to an alphanumeric value, OR escape with backslashes inside the quoted value: `Environment="MULE_LOGGER_PASSWORD=My\$Pass\!word"` |
| Editing the package-shipped unit file directly works once, then breaks after `apt-get upgrade` | `sudo systemctl edit --full` (or hand-editing `/lib/systemd/system/otelcol-contrib.service`) is overwritten by `dpkg` | Move the change into a drop-in: `sudo systemctl edit otelcol-contrib` (no flags) — drop-ins survive package upgrades |
| Documents arrive but the time filter in Discover hides them | Data view's time field does not match the field name the exporter emits | Edit the data view and pick the field that actually exists on the document |
| Property `mule.openTelemetry.logging.exporter.enabled=true` but app starts on a runtime version <4.11.0 | Direct Telemetry Stream requires Mule 4.11.0+ | Bump the runtime version in the deployment to 4.11.0 |
| Property accepted but Anypoint Monitoring "Telemetry Exporter" tab is gone | Expected — the docs explicitly say Anypoint Monitoring's OTel features become unavailable when Direct Telemetry Stream is on | Not a bug; pick one path per environment |
| Org subscription does not show CH2/RTF Properties save the OTel keys | Subscription tier below Advanced/Titanium | Confirm with org admin; otherwise fall back to the Part 4 path |

---

## What We Covered

- We took the Mule 4.11 hello-world from Part 4, **removed its `log4j2.xml`**, and republished it to Exchange under a new artifact id.
- We provisioned a third EC2 instance running **OpenTelemetry Collector Contrib** with an OTLP/HTTP receiver and an Elasticsearch exporter pointed at `mule-logs`.
- We deployed the artifact to **CloudHub 2.0** and **Runtime Fabric** with the **Direct Telemetry Stream — Logs Exporter** turned on via six Runtime Manager properties.
- We hit both endpoints and saw OTLP log records flow runtime → collector → `mule-logs`, distinguishable by `Resource.service.name`.
- We compared the two log-shipping paths and noted the trade-offs — Mule version, subscription tier, who owns the destination config.

We now have **two independent ways** to put Mule logs into Elastic running side-by-side, and an OTel Collector ready to receive its second signal.

> ➡️ **Next up:** Part 6 takes a step back from the tutorials and goes deep on *why* we need that OTel Collector at all — what OpenTelemetry support in Mule actually means, why OTLP/protobuf cannot be POSTed straight to Elasticsearch's `_doc` endpoint, and the full set of alternatives (Log4j2, Filebeat, Logstash, Elastic APM Server, native OTLP). It is a deep-dive, not a tutorial. Part 7 then turns on the **Tracer Exporter** with `mule.openTelemetry.tracer.exporter.*`, points it at the same collector we built here, and adds a `traces` pipeline that exports to the `mule-traces` data stream. Because we already set `mule.put.trace.id.and.span.id.in.mdc=true` here in Part 5, every log line in `mule-logs` will arrive with the matching `trace.id` from day one — that is what unlocks the unified dashboard in Part 8.

---

## References

- [OpenTelemetry support for Mule runtime — MuleSoft Docs](https://docs.mulesoft.com/mule-runtime/latest/otel-support)
- [Direct Telemetry Stream for Traces Configuration — MuleSoft Docs](https://docs.mulesoft.com/mule-runtime/latest/otel-support#direct-telemetry-stream-for-traces-configuration)
- [Setting Property Values in Runtime Manager — MuleSoft Docs](https://docs.mulesoft.com/runtime-manager/configuring-properties#setting-properties-values-in-runtime-manager)
- [OpenTelemetry Collector — Elasticsearch Exporter](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/elasticsearchexporter)
- [OpenTelemetry Collector — OTLP Receiver](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver)
- [OTLP/HTTP Specification — OpenTelemetry](https://opentelemetry.io/docs/specs/otlp/#otlphttp)
- [Mule 4.11 Release Notes — MuleSoft Docs](https://docs.mulesoft.com/release-notes/mule-runtime/mule-4.11.0-release-notes)
