---
title: "Hello-World Mule App — Log4j2 HTTP Appender to Elasticsearch — Part 4"
slug: "mule-hello-world-log4j2-http-appender-elasticsearch"
description: "Build a minimal Mule 4.11 app that ships logs to the HTTP-only Elasticsearch cluster from Part 1 with the Log4j2 HttpAppender — Basic Auth via the mule-logger user, no truststore needed."
author: "Gonzalo Marcos"
date: 2026-06-08
status: not validated
lang: en
category: observability
tags:
  - mule-runtime
  - log4j
  - logging
  - elasticsearch
  - kibana
  - english
  - series
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 4
type: tutorial
difficulty: intermediate
read_time: 15
mule_version: "4.11"
platform:
  - standalone
  - anypoint-platform
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![Standalone](https://img.shields.io/badge/Platform-Standalone-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 4](https://img.shields.io/badge/Part-4-16a085) ![15 min](https://img.shields.io/badge/Read_Time-15_min-lightgrey)

# Hello-World Mule App — Log4j2 HTTP Appender to Elasticsearch — Part 4

In [Part 1](#) we installed Elasticsearch on AWS with Basic Auth over plain HTTP, in [Part 2](#) we paired Kibana with it, and in [Part 3](#) we created the `mule-logs` index, the `mule-logs-writer` role, and the `mule-logger` service user. The cluster is up, scoped, and waiting. The only thing missing is the data.

In this tutorial we will build the smallest possible Mule app on **Mule Runtime 4.11** — an HTTP listener that logs `"Hello World"` and returns a JSON response — and wire its logging layer to Elasticsearch through Log4j2's `HttpAppender` over **plain HTTP**. Every time we hit the endpoint, a log document will land in the `mule-logs` index and we will see it in Kibana's Discover.

Because the Elasticsearch cluster speaks HTTP (not HTTPS) on the data plane, there is no truststore to set up, no CA to import, no `keytool` step, no PKIX errors. The Log4j2 appender just opens a TCP connection, sends a `POST`, attaches a Basic Auth header, and we are done. That is the whole point of the HTTP-only profile we built in Part 1.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 3 — Initial Elasticsearch Config: Indexes, Roles, and Users for Logs + Traces](#) · **Next:** Part 5 — Hello-World Mule App on CloudHub 2.0 / RTF: Direct Stream Logging (No Log4j2)

> [!WARNING]
> **HTTP-only, demo-grade.** Every log line we ship in this tutorial — including the `Authorization: Basic ...` header — crosses the wire **unencrypted**. Acceptable inside a private VPC with a tight security group; **not** appropriate for shared networks. The sister series *Elastic Stack for MuleSoft Observability* covers the TLS variant of this exact appender, including the JVM truststore and bundled-JKS strategies.

---

## What We Will Cover

- Build a minimal Mule 4.11 hello-world app — one HTTP listener, one Logger, one Set Payload.
- Generate a Basic Auth header from the `mule-logger` credentials and inject it via env var.
- Wire `log4j2.xml` so every event POSTs to `http://<elastic>:9200/mule-logs/_doc` as a JSON document.
- Run the app from Anypoint Studio, hit the endpoint, and watch the documents appear in Kibana Discover.
- Skip the truststore plumbing entirely — that is the trade-off we made in Part 1, and Part 4 is where it pays off.

---

## Prerequisites

Before we start, we will need:

- A working Elasticsearch + Kibana from [Parts 1–3](#), reachable over plain HTTP from the dev machine.
- The `mule-logger` user credentials from [Part 3](#).
- A development machine with **Anypoint Studio 7.x targeting Mule 4.11** (or just Maven and the Mule Maven Plugin) and **JDK 17**.
- Network reachability from the dev machine to the Elasticsearch host on port `9200/tcp` — open the security group from our admin IP if needed.
- `curl` and `base64` on the path (default on macOS, Linux, and Git Bash for Windows).

We will export the values we will reuse below into the shell. Replace the placeholders with our actual values:

```bash
export ELASTIC_HOST="10.0.1.20"               # the Part 1 ES host's private IP
export MULE_LOGGER_PASSWORD='<MULE_LOGGER_PASSWORD>'
```

We will sanity-check the cluster is reachable and the credentials work before touching Studio:

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://$ELASTIC_HOST:9200/mule-logs/_count?pretty"
```

We expect a JSON object with `"count": 1` (the document we wrote in Part 3). If we see `401`, the password is wrong; if we see `connection refused` or a timeout, the security group or the bind address from Part 1 needs another look.

---

## Why HTTP and Not HTTPS Here?

We are intentionally trading TLS for simplicity in this series. That makes Log4j2 wiring trivial, but it also means we are **not** doing the things the production-grade variant does:

| Concern | HTTP-only (this post) | HTTPS variant (sister series) |
| --- | --- | --- |
| Log4j2 `<Http>` URL | `http://...` | `https://...` |
| `<SslConfiguration>` block | absent | required |
| JVM truststore / bundled JKS | not used | needed (`keytool`, `cacerts`, or app-local JKS) |
| Wire encryption | none | TLS 1.2/1.3 |
| `Authorization` header on the wire | base64-encoded, **unencrypted** | base64-encoded, **inside TLS** |
| CA rotation runbook | n/a | required |

The whole point of this series is to keep the Mule side of the configuration to the bare minimum so we can move on to the parts that *only* this series covers — the Direct Stream variant in Part 5 and distributed traces in Part 6.

---

## Overview

```
┌──────────────────────────────────────────────────────┐
│  Dev machine (Anypoint Studio, Mule 4.11)            │
│                                                      │
│   ┌──────────────────────────────────────────┐       │
│   │  hello-world-mule app                    │       │
│   │                                          │       │
│   │  ┌────────┐   ┌─────────┐   ┌─────────┐  │       │
│   │  │  HTTP  │──▶│ Logger  │──▶│  Set    │  │       │
│   │  │ :8081  │   │  INFO   │   │ Payload │  │       │
│   │  └────────┘   └────┬────┘   └─────────┘  │       │
│   │                    │ Log4j2 HttpAppender │       │
│   └────────────────────┼─────────────────────┘       │
└────────────────────────┼─────────────────────────────┘
                         │ HTTP POST + Basic Auth
                         ▼
        http://10.0.1.20:9200/mule-logs/_doc
                  ┌──────────────┐
                  │ Elasticsearch│
                  │   mule-logs  │
                  └──────────────┘
```

---

## Table of Contents

1. [Step 1 — Create the `hello-world-mule` Project](#step-1--create-the-hello-world-mule-project)
2. [Step 2 — Build the Hello-World Flow](#step-2--build-the-hello-world-flow)
3. [Step 3 — Encode the Basic Auth Credentials](#step-3--encode-the-basic-auth-credentials)
4. [Step 4 — Configure `log4j2.xml` with the Elastic Appender](#step-4--configure-log4j2xml-with-the-elastic-appender)
5. [Step 5 — Inject the Env Vars into the Studio Run Configuration](#step-5--inject-the-env-vars-into-the-studio-run-configuration)
6. [Step 6 — Run the App](#step-6--run-the-app)
7. [Step 7 — Trigger the Flow and Verify in Kibana](#step-7--trigger-the-flow-and-verify-in-kibana)
8. [Verification](#verification)
9. [Troubleshooting](#troubleshooting)

---

### Step 1 — Create the `hello-world-mule` Project

In Anypoint Studio:

1. **File → New → Mule Project**.
2. **Project Name:** `hello-world-mule`.
3. **Runtime:** Mule **4.11.x** (the minimum for this series).
4. Click **Finish**.


Studio creates the standard layout we will work in:

```text
hello-world-mule/
├── pom.xml
├── mule-artifact.json
└── src/main/
    ├── mule/                  ← we will edit hello-world.xml here
    └── resources/
        └── log4j2.xml         ← we will replace this
```

If we are not using Studio, the same layout is what `mvn archetype:generate` produces with the Mule application archetype.

---

### Step 2 — Build the Hello-World Flow

We will build the simplest possible flow: an HTTP listener on `:8081/hello`, a Logger that writes an INFO line, and a Set Payload that returns a JSON response.

The Studio path:

1. From the **Mule Palette**, drag an **HTTP Listener** onto the canvas.
2. Configure the connector with `host: 0.0.0.0`, `port: 8081`, **Path:** `/hello`.
3. Drag a **Logger** after the listener. **Message:** `Hello World - request received`. **Level:** `INFO`.
4. Drag a **Set Payload**. **Value:** `'{"message": "Hello World"}'`. **Mime type:** `application/json`.

Or paste this directly into `src/main/mule/hello-world.xml`:

📄 `src/main/mule/hello-world.xml`

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
        <http:listener-connection host="0.0.0.0" port="8081"/>
    </http:listener-config>

    <flow name="hello-world-flow">
        <http:listener config-ref="HTTP_Listener_config" path="/hello"/>
        <logger level="INFO" message="Hello World - request received"/>
        <set-payload value='#[output application/json --- { "message": "Hello World" }]'/>
    </flow>

</mule>
```

📄 Full file: [./assets/hello-world-mule/src/main/mule/hello-world.xml](./assets/hello-world-mule/src/main/mule/hello-world.xml)

> [!NOTE]
> *Screenshot — Anypoint Studio canvas showing the three-step flow (Listener → Logger → Set Payload).*

---

### Step 3 — Encode the Basic Auth Credentials

The Log4j2 `HttpAppender` does not have a first-class auth setting. It accepts arbitrary HTTP headers, so we add an `Authorization` header with a base64-encoded `<user>:<password>` payload — exactly what HTTP Basic Auth is on the wire.

We will generate the encoded value — we will paste it into the launch configuration in Step 5 so the credentials never end up in `log4j2.xml`:

```bash
echo -n "mule-logger:$MULE_LOGGER_PASSWORD" | base64
```

We will keep the output handy as `<ELASTIC_AUTH_BASE64>` for now:

```bash
export ELASTIC_AUTH='<ELASTIC_AUTH_BASE64>'
```

> [!WARNING]
> Putting a base64 of `user:password` in any config file (or env var) is one notch better than hardcoding — it keeps the cleartext out of `log4j2.xml`, but base64 is encoding, not encryption. Anyone with read access can decode it. In production we will load the password from Mule Secure Properties, AWS Secrets Manager, or HashiCorp Vault, and inject it at runtime.

> [!IMPORTANT]
> On HTTP-only the encoded credential is exposed twice over: in the launch configuration on disk **and** in the `Authorization` header on the wire. Two reasons we accept it here: this cluster is demo-only, and we have already accepted plain-HTTP traffic in the security model from Part 1. The Mule-side wire-up is identical to the HTTPS variant — the only difference is the `http://` URL we will use in Step 4.

---

### Step 4 — Configure `log4j2.xml` with the Elastic Appender

We will replace the contents of `src/main/resources/log4j2.xml` with the version below. It keeps the default `RollingFile` appender (so we still get an on-disk log) and adds an `Http` appender that POSTs every event as a JSON document to Elasticsearch over plain HTTP.

📄 `src/main/resources/log4j2.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<Configuration status="WARN" name="hello-world-mule">

    <Appenders>
        <!-- Default Mule app log file -->
        <RollingFile name="file"
                     fileName="${sys:mule.home}${sys:file.separator}logs${sys:file.separator}hello-world-mule.log"
                     filePattern="${sys:mule.home}${sys:file.separator}logs${sys:file.separator}hello-world-mule-%i.log">
            <PatternLayout pattern="[%d{ISO8601}] %-5p [%t] %c: %m%n"/>
            <DefaultRolloverStrategy max="10"/>
            <Policies>
                <SizeBasedTriggeringPolicy size="10 MB"/>
            </Policies>
        </RollingFile>

        <!-- Ship every event to Elasticsearch as a JSON document -->
        <Http name="Elastic"
              url="http://${sys:ELASTIC_HOST}:9200/mule-logs/_doc">
            <Property name="Content-Type" value="application/json"/>
            <Property name="Authorization" value="Basic ${sys:ELASTIC_AUTH}"/>
            <JsonLayout properties="true"
                        compact="true"
                        eventEol="true"
                        stacktraceAsString="true"
                        includeTimeMillis="true">
                <KeyValuePair key="@timestamp" value="$${date:yyyy-MM-dd'T'HH:mm:ss.SSSXXX}"/>
                <KeyValuePair key="service.name" value="hello-world-mule"/>
            </JsonLayout>
        </Http>
    </Appenders>

    <Loggers>
        <AsyncRoot level="INFO">
            <AppenderRef ref="file"/>
            <AppenderRef ref="Elastic"/>
        </AsyncRoot>
    </Loggers>

</Configuration>
```

📄 Full file: [./assets/hello-world-mule/src/main/resources/log4j2.xml](./assets/hello-world-mule/src/main/resources/log4j2.xml)

A few things worth knowing about this configuration:

| Element                                 | Purpose                                                                                                     |
| --------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `<Http url="http://...">`               | Log4j2's built-in HTTP appender. Plain HTTP — no `<SslConfiguration>` block needed.                         |
| `${sys:ELASTIC_HOST}` / `${sys:ELASTIC_AUTH}` | Resolved from JVM **system properties** set with `-D...` — *not* OS environment variables. ACB's `mule-xml-debugger` does not propagate `env` to the embedded Mule JVM, so we use `-M-D` flags instead and read them with `${sys:...}`. |
| `Property: Authorization`               | Becomes a request header. The base64 `user:password` is read from the system property at startup.           |
| `<JsonLayout>`                          | Renders each event as a single-line JSON document — Elasticsearch will accept it on `_doc`.                 |
| `compact="true" eventEol="true"`        | One JSON object per line, no pretty-print.                                                                  |
| `<KeyValuePair key="@timestamp" ...>`   | Adds an ISO-8601 timestamp on the event. This is what makes the **Mule Logs** data view's time filter work. |
| `<KeyValuePair key="service.name" ...>` | Tags every event with the app name — Part 7's dashboard groups by this field.                               |
| `<AsyncRoot>`                           | Decouples log emission from the flow thread. The HTTP POST happens off the request path.                    |

> [!IMPORTANT]
> The `HttpAppender` does **not** batch — every log line is a separate HTTP request. That is fine for a hello-world; for a busy production app it is wasteful and we will switch to batching (or Filebeat as a side-car) in a later post.

> [!TIP]
> The plain `JsonLayout` emits a `timeMillis` field but no `@timestamp`. By adding the `<KeyValuePair key="@timestamp" ...>` we feed the Kibana data view's time filter directly. The sister series swaps `JsonLayout` for `JsonTemplateLayout` with the ECS template, which adds a much richer set of fields — overkill for this demo.

---

### Step 5 — Inject the Values via the Launch Configuration

A shell `export` only affects the shell that launched it. The IDE runs the Mule app in its own JVM and does not inherit the parent shell's env vars unless we tell it to. We will pass the two values as JVM **system properties** through the launch configuration, which is what `${sys:ELASTIC_HOST}` and `${sys:ELASTIC_AUTH}` in `log4j2.xml` resolve from.

#### Option A — Anypoint Code Builder (recommended)

ACB is VS Code under the hood, so the launch configuration lives in `.vscode/launch.json`. The `mule-xml-debugger` adapter accepts JVM system properties via `mule.runtime.args`. We will use `-M-D<name>=<value>` flags — the `-M` prefix tells the Mule launcher to pass the flag through to the embedded JVM as a system property.

> [!IMPORTANT]
> The `mule-xml-debugger` adapter does **not** forward an `"env": {}` block to the spawned Mule runtime — anything we put there is silently ignored. Use `mule.runtime.args` with `-M-D...` flags. We learned this the hard way; do not waste time on `env`.

1. In ACB, open the project.
2. Open the **Run and Debug** panel (left sidebar).
3. Click the gear icon to open `.vscode/launch.json` (or click *create a launch.json file* and pick **Mule** if the file does not exist yet).
4. Add `-M-DELASTIC_HOST=...` and `-M-DELASTIC_AUTH=...` to the run configuration's `mule.runtime.args`:

📄 `.vscode/launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "mule-xml-debugger",
      "request": "launch",
      "name": "Run Mule Application",
      "noDebug": true,
      "mule.projects": ["${workspaceFolder}"],
      "mule.runtime.args": "${config:mule.runtime.defaultArguments} -M-DELASTIC_HOST=ec2-X-XX-XXX-XX.eu-central-1.compute.amazonaws.com -M-DELASTIC_AUTH=********************"
    },
    {
      "type": "mule-xml-debugger",
      "request": "launch",
      "name": "Debug Mule Application",
      "mule.projects": ["${workspaceFolder}"],
      "mule.runtime.args": "${config:mule.runtime.defaultArguments}"
    }
  ]
}
```

Replace `********************` with the base64 string we generated in Step 3, and `ec2-X-XX-XXX-XX...` with the actual hostname or private IP of the Elasticsearch instance from Part 1. Save the file.

> [!WARNING]
> `.vscode/launch.json` lives in the workspace and is easy to commit to Git by accident — together with the base64 of `mule-logger:<password>`, which decodes back to the cleartext password in one `base64 -d`. Add `.vscode/launch.json` to `.gitignore`, or replace the literal value with `${env:ELASTIC_AUTH_BASE64}` and `export` the variable in the shell that launches ACB.

> [!NOTE]
> *Screenshot — ACB's Run and Debug panel showing the **Run Mule Application** configuration selected, and `.vscode/launch.json` open in the editor with the two `-M-D` flags highlighted.*

#### Option B — Anypoint Studio

Studio does not use `launch.json`. We set system properties on the Run Configuration's **Arguments** tab instead.

1. **Run → Run Configurations…**
2. Select **Mule Application → hello-world-mule** (Studio creates this entry automatically the first time we run the app; if it is missing, run it once and come back).
3. Open the **Arguments** tab → in **VM arguments** add:
   ```text
   -DELASTIC_HOST=ec2-X-XX-XXX-XX.eu-central-1.compute.amazonaws.com
   -DELASTIC_AUTH=********************
   ```
4. Click **Apply**, then **Close**.

> [!TIP]
> Studio also exposes an **Environment** tab — that one *does* propagate to the JVM. If we prefer real env vars over system properties, we can use that tab and switch `${sys:...}` back to `${env:...}` in `log4j2.xml`. We picked `-D` system properties here only because it is the path that works on **both** ACB and Studio with the same `log4j2.xml`.

#### Option C — Maven CLI

```bash
mvn clean package mule:run \
  -DELASTIC_HOST=ec2-X-XX-XXX-XX.eu-central-1.compute.amazonaws.com \
  -DELASTIC_AUTH='<ELASTIC_AUTH_BASE64>'
```

The `-D` flags become JVM system properties for the embedded runtime, just like the IDE paths above.

#### Restart, do not just reload

A running JVM caches system properties at boot. After editing `launch.json` (or the Studio Arguments tab), **stop the running app and re-launch it** — a workspace reload is not enough.

---

### Step 6 — Run the App

In Studio: right-click the project → **Run As → Mule Application**.

In the console we will look for two healthy lines:

```text
INFO  ... Mule is up and kicking (your view, my friend)
INFO  ... Started app 'hello-world-mule'
```

If the app fails to start, check the console for stack traces. The most common Log4j2-related failures show up as a single warning at the very top of the boot output — `Http Appender failed to initialize` or similar. We will address those in Troubleshooting.

> [!NOTE]
> *Screenshot — Studio console showing both `Mule is up and kicking` and `Started app 'hello-world-mule'`.*

---

### Step 7 — Trigger the Flow and Verify in Kibana

We will hit the endpoint a few times so we have something to look at:

```bash
curl -s http://localhost:8081/hello
curl -s http://localhost:8081/hello
curl -s http://localhost:8081/hello
```

Each call should return:

```json
{"message":"Hello World"}
```

Now switch to Kibana → **Analytics → Discover** → select the **Mule Logs** data view from Part 3.

We expect to see at least three documents (one per request), each with fields like:

| Field | Example value |
| --- | --- |
| `@timestamp` | `2026-06-08T10:14:23.512+00:00` |
| `level` | `INFO` |
| `loggerName` | `org.mule.runtime.core.internal.processor.LoggerMessageProcessor` |
| `message` | `Hello World - request received` |
| `service.name` | `hello-world-mule` |
| `thread` | `[hello-world-mule].uber.04` |
| `timeMillis` | `1733415892000` |

Click any row to expand it and confirm `message` matches what we logged in the flow. The time filter at the top right (default **Last 15 minutes**) should already include our requests because of the `@timestamp` field we added in Step 4.

![](Hello-World%20Mule%20App%20—%20Log4j2%20HTTP%20Appender%20to%20Elasticsearch%20—%20Part%204.png)

---

## Verification

We will run three checks to confirm the pipeline is wired end-to-end.

**1. The Mule app's local log file shows the same events.**

```bash
tail -n 20 ~/AnypointStudio/studio-workspace/.mule/apps/hello-world-mule/logs/hello-world-mule.log
```

(Path varies by OS and Studio workspace.) We expect to see one `Hello World - request received` line per request.

**2. Elasticsearch shows the same documents.**

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://$ELASTIC_HOST:9200/mule-logs/_count?pretty"
```

The count should be at least three (or whatever number of requests we made), plus the document we wrote in Part 3.

**3. The Logger level survives the round trip.**

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://$ELASTIC_HOST:9200/mule-logs/_search?q=level:INFO&pretty"
```

We expect every hit to have `"level": "INFO"`. If we change the Logger component in Studio to `level: WARN` and hit the endpoint again, the new documents should have `"level": "WARN"`.


---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Mule console shows `received plaintext http traffic on an https channel` from Elastic | The appender URL is `https://...` but the cluster is HTTP-only | In `log4j2.xml`, set `<Http url="http://...">` (no `s`) |
| Mule console shows `unable to find protocol handler for "https"` | We tried HTTPS without the right Log4j2 SSL plugin — but we are HTTP-only here | Same as above — flip the URL back to `http://...` |
| No documents appear in `mule-logs`, no errors | `Authorization` header is wrong, or `ELASTIC_AUTH` is empty | Echo the env var inside Studio's Run Configuration; re-run `echo -n "mule-logger:$MULE_LOGGER_PASSWORD" \| base64` and compare |
| Documents appear but Discover says "No results" with the time filter on | `@timestamp` is missing because the `<KeyValuePair>` was dropped from the layout | Re-check `log4j2.xml`; make sure the `<KeyValuePair>` block is inside `<JsonLayout>`, not a sibling |
| `403 Forbidden` in Studio console from the appender | The `mule-logger` user lost its `write` privilege, or it is hitting `mule-traces` by accident | `GET /_security/user/mule-logger` and `GET /_security/role/mule-logs-writer`; re-check the URL in `log4j2.xml` |
| `401 Unauthorized` in the appender's debug output | Wrong credentials in `ELASTIC_AUTH`, or `MULE_LOGGER_PASSWORD` was rotated | Regenerate the base64 from a known-good password; verify with `curl -u mule-logger:... http://$ELASTIC_HOST:9200` |
| App starts but logs are not shipped — no errors either | `<AsyncRoot>` is silently dropping events because the queue is full / disruptor backpressure | Confirm the Elastic host is reachable from the dev machine; raise the async logger queue size or temporarily switch to `<Root>` (synchronous) for debugging |
| Mule cannot resolve `${sys:ELASTIC_HOST}` (URL becomes `http://:9200/...`) | The values were set as `env` instead of `-M-D` system properties | In ACB, use `mule.runtime.args` with `-M-D...` (Step 5 Option A); the `mule-xml-debugger` adapter ignores the `env` block. In Studio, use the **Arguments** tab's VM arguments. |
| Logs were going through, then suddenly stop after editing `launch.json` | The running JVM is still using the old (unset) values | Stop the running app from the Run and Debug panel and re-launch; a reload alone does not pick up new system properties |
| Connection from the dev machine times out | Security group does not allow `9200/tcp` from our admin IP, or ES bound to `127.0.0.1` | Open the SG rule on the ES instance; revisit Part 1 Step 6 |
| `document_parsing_exception` from Elasticsearch after also wiring Part 5 against the same `mule-logs` index | This Part 4 appender's `JsonLayout` writes `thread` as a flat string; Part 5's OTel exporter writes `thread.id` and `thread.name` as a nested object. Whichever writer ingests first locks the dynamic mapping, and the other side's documents are rejected. | Pick a strategy: (1) **separate indexes per writer** — write Part 4 to `mule-logs-log4j` and Part 5 to `mule-logs-otlp`; the Part 8 dashboard runs on `mule-logs-*`; or (2) **one shape, one index** — switch Part 5's `elasticsearch/logs` exporter to `mapping.mode: ecs` and migrate this `log4j2.xml` to `JsonTemplateLayout` with `eventTemplateUri="classpath:EcsLayout.json"` so both writers emit the same ECS field names. |

> [!TIP]
> Log4j2's HttpAppender swallows transport errors by default. To see what it is actually doing on the wire, set the Log4j2 status logger to `DEBUG`: change `<Configuration status="WARN" ...>` to `<Configuration status="DEBUG" ...>` at the top of `log4j2.xml`. Revert it once we have diagnosed the issue — it is loud.

---

## What We Covered

- We built a hello-world Mule 4.11 app with one HTTP listener, one Logger, and one Set Payload.
- We generated a Basic Auth header from the `mule-logger` credentials and injected it into Studio's Run Configuration as an env var, keeping the cleartext out of `log4j2.xml`.
- We added a Log4j2 `HttpAppender` that POSTs every event to `mule-logs/_doc` over **plain HTTP** — no truststore, no `<SslConfiguration>`, no `keytool`.
- We added `@timestamp` and `service.name` to the JSON layout so the Kibana data view's time filter and the Part 7 dashboard work without extra wiring.
- We hit the endpoint and confirmed the documents land in `mule-logs` and surface in Kibana's Discover view.

We now have a working end-to-end pipeline: HTTP request → Mule flow → Log4j2 → Elasticsearch → Kibana. Every Logger statement we add to a Mule app, anywhere in the project, will start showing up in Discover automatically — and we did it without touching a certificate.

> ➡️ **Next up:** In Part 5 we will deploy a sibling hello-world app to **CloudHub 2.0 / Runtime Fabric** without any custom `log4j2.xml`. Instead of the HttpAppender, we will use Direct Stream — Anypoint Runtime Manager's native log forwarding configured via deployment **properties**. Same payload, different shipping mechanism, identical Kibana view.

---

## References

- [HttpAppender — Log4j2 Manual](https://logging.apache.org/log4j/2.x/manual/appenders.html#HttpAppender)
- [JsonLayout — Log4j2 Manual](https://logging.apache.org/log4j/2.x/manual/layouts.html#JSONLayout)
- [Async Loggers — Log4j2 Manual](https://logging.apache.org/log4j/2.x/manual/async.html)
- [Configure Logging in Mule 4 — MuleSoft Docs](https://docs.mulesoft.com/mule-runtime/latest/logging-in-mule)
- [Index API — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-index_.html)
- [Mule 4.11 Release Notes — MuleSoft Docs](https://docs.mulesoft.com/release-notes/mule-runtime/mule-4.11.0-release-notes)
- [Elastic Common Schema (ECS)](https://www.elastic.co/guide/en/ecs/current/index.html)
