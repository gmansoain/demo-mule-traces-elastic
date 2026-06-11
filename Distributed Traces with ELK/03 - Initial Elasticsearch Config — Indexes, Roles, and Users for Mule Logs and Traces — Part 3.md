---
title: Initial Elasticsearch Config — Indexes, Roles, and Users for Mule Logs and Traces — Part 3
slug: elasticsearch-indexes-roles-users-mule-logs-traces
description: Create least-privilege roles and service users for Mule logs and traces, and Kibana data views over both, on the HTTP-only cluster from Parts 1 and 2.
author: Gonzalo Marcos
date: 2026-06-08
status: validated
lang: en
category: observability
tags:
  - elasticsearch
  - kibana
  - logging
  - secrets-management
  - best-practices
  - english
  - series
series: Sending Logs and Distributed Traces to Elastic
series_part: 3
type: tutorial
difficulty: intermediate
read_time: 14
platform:
  - aws
  - anypoint-platform
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![AWS](https://img.shields.io/badge/AWS-FF9900?logo=amazonaws&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 3](https://img.shields.io/badge/Part-3-16a085) ![14 min](https://img.shields.io/badge/Read_Time-14_min-lightgrey)

# Initial Elasticsearch Config — Indexes, Roles, and Users for Mule Logs and Traces — Part 3

In [Part 1](#) we installed Elasticsearch on AWS with Basic Auth over plain HTTP, and in [Part 2](#) we paired Kibana with it from a separate EC2 instance. The stack is up and we can log in as `elastic` — but before we point any Mule app at the cluster we need to lay groundwork so the apps never see the superuser password.

In this tutorial we will create **two** indexes (`mule-logs` for application logs and `mule-traces` for distributed-trace spans), **two** least-privilege roles (`mule-logs-writer` and `mule-traces-writer`), and **two** service users (`mule-logger` and `mule-tracer`) — one credential per concern, scoped to one destination. We will then add Kibana data views over both so a single dashboard in Part 7 can correlate logs and traces in the same screen.

By the end of the post Postman will be able to write a log document and a trace document using the new service users, and Kibana's Discover view will see both. From there, swapping Postman for a Mule HTTP Request connector (Part 4) or the OpenTelemetry Collector's Elasticsearch exporter (Part 6) is a configuration change.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 2 — Installing Kibana on Ubuntu 26.04 on a Separate AWS EC2 Instance](#) · **Next:** Part 4 — Hello-World Mule App: Log4j2 HTTP Appender → Elasticsearch

> [!WARNING]
> **HTTP-only, demo-grade.** Every API call in this post crosses the wire over **plain HTTP**, including the Basic Auth credentials we are about to create. Acceptable inside a private VPC with a tight security group; **not** appropriate for shared networks. Treat the `mule-logger` and `mule-tracer` passwords as exposable secrets and rotate them when the demo is retired.

---

## What We Will Cover

- Create dedicated indexes for Mule logs and Mule traces.
- Create two custom roles, each scoped to a single index pattern with `read` + `write` only.
- Create two service users, each bound to exactly one role — no cross-purpose credentials.
- Test the credentials end-to-end: write a document, fail with bad credentials, fail with the *wrong* user (denied across concerns), read the documents back.
- Create Kibana data views over both indexes so we can browse documents in Discover and prepare for the unified dashboard.

---

## Prerequisites

Before we start, we will need:

- A working Elasticsearch node from [Part 1](#) — we will keep using `10.0.1.20` as its private IP.
- A working Kibana from [Part 2](#) on `10.0.1.21`, reachable in a browser.
- The `elastic` superuser password from Part 1 captured in `$ELASTIC_PASSWORD`.
- Postman, Insomnia, or any HTTP client that supports Basic Auth — we will show both Postman and `curl`.
- `jq` on the Elasticsearch host (handy for the verification step).

We will confirm both services are healthy before continuing:

```bash
curl -u elastic:$ELASTIC_PASSWORD http://10.0.1.20:9200/_cluster/health?pretty
curl http://10.0.1.21:5601/api/status | jq '.status.overall'
```

We expect a `green` cluster and Kibana reporting `available`.

---

## Why Two Indexes, Two Roles, Two Users?

The original "single index + single user" pattern works for one signal but breaks the moment we add a second. Three reasons we are splitting from day one:

1. **Blast radius per signal.** A leaked `mule-tracer` password gives an attacker write access to spans, not to logs — and vice versa. One stolen credential cannot poison the other dataset.
2. **Auditability.** When a stray document shows up at 3 AM, knowing it was written by `mule-logger` versus `mule-tracer` versus `elastic` versus `kibana_system` cuts the investigation down to seconds.
3. **Independent retention.** Logs and traces have different volume profiles and different value-over-time. Splitting them now lets us attach different ILM policies later without re-bucketing data.

We will follow the principle of **least privilege**: an index that holds exactly one signal, a role that grants exactly the permissions needed to write that signal, and a user dedicated to that role.

| Decision | Options Considered | Chosen | Rationale |
| --- | --- | --- | --- |
| Index layout | Single `mule-events` for both / two indexes / two data streams | Two plain indexes (`mule-logs`, `mule-traces`) | Clean separation, both can grow into data streams later (Part 6 will add a template for traces) |
| Role granularity | One `mule-writer` role on both / two roles, one per index | Two roles | Each app gets only what it needs; revocation does not affect the other signal |
| User granularity | One `mule` user with both roles / two users | Two users | A leaked credential only loses one dataset, not both |

---

## Overview

```
┌────────────────────────────────────────────────────────────────────┐
│  Elasticsearch  (Part 1, HTTP-only, Basic Auth)                    │
│                                                                    │
│  ┌──────────────┐  ┌──────────────────────────┐                    │
│  │ mule-logs    │◀─│ Role: mule-logs-writer   │◀─┐                 │
│  │ index        │  │  read,write on mule-logs │  │                 │
│  └──────────────┘  └──────────────────────────┘  │                 │
│                                                  │                 │
│                                ┌─────────────────┴────────────┐    │
│                                │ User: mule-logger            │    │
│                                │   role: mule-logs-writer     │    │
│                                └──────────────────────────────┘    │
│                                                                    │
│  ┌──────────────┐  ┌──────────────────────────┐                    │
│  │ mule-traces  │◀─│ Role: mule-traces-writer │◀─┐                 │
│  │ index        │  │  read,write on mule-traces│ │                 │
│  └──────────────┘  └──────────────────────────┘  │                 │
│                                                  │                 │
│                                ┌─────────────────┴────────────┐    │
│                                │ User: mule-tracer            │    │
│                                │   role: mule-traces-writer   │    │
│                                └──────────────────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
        ▲                                ▲
        │                                │
   Mule app (Part 4/5)              OTel Collector (Part 6)
```

---

## Table of Contents

1. [Step 1 — Create the `mule-logs` and `mule-traces` Indexes](#step-1--create-the-mule-logs-and-mule-traces-indexes)
2. [Step 2 — Create the `mule-logs-writer` and `mule-traces-writer` Roles](#step-2--create-the-mule-logs-writer-and-mule-traces-writer-roles)
3. [Step 3 — Create the `mule-logger` and `mule-tracer` Users](#step-3--create-the-mule-logger-and-mule-tracer-users)
4. [Step 4 — Test Each User Can Write to Its Index](#step-4--test-each-user-can-write-to-its-index)
5. [Step 5 — Test Authentication Fails with Bad Credentials](#step-5--test-authentication-fails-with-bad-credentials)
6. [Step 6 — Test Cross-Concern Access Is Denied](#step-6--test-cross-concern-access-is-denied)
7. [Step 7 — Test Each User Can Read Its Index Back](#step-7--test-each-user-can-read-its-index-back)
8. [Step 8 — Create the Kibana Data Views](#step-8--create-the-kibana-data-views)
9. [Step 9 — Browse the Documents in Discover](#step-9--browse-the-documents-in-discover)
10. [Verification](#verification)
11. [Troubleshooting](#troubleshooting)

---

### Step 1 — Create the `mule-logs` and `mule-traces` Indexes

An index in Elasticsearch is the data structure that stores and organizes documents — the rough equivalent of a database in a relational system. We want one dedicated index per signal so we can scope permissions, retention, and lifecycle policies independently.

We can do this from the Kibana UI or from the REST API. Pick one — they have the same effect.

#### Option A — Kibana UI

1. From the Kibana home page, click the menu button on the top left and go to **Stack Management**.
2. From the left panel, click **Index Management**.
3. Click **Create index**, name it `mule-logs`, and click **Create index**.
4. Repeat for `mule-traces`.

![](Initial%20Elasticsearch%20Config%20—%20Indexes,%20Roles,%20and%20Users%20for%20Mule%20Logs%20and%20Traces%20—%20Part%203.png)

#### Option B — Elasticsearch REST API

```bash
curl -X PUT -u elastic:$ELASTIC_PASSWORD http://10.0.1.20:9200/mule-logs
curl -X PUT -u elastic:$ELASTIC_PASSWORD http://10.0.1.20:9200/mule-traces
```

We expect a response like this for each index:

```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "mule-logs"
}
```

> [!TIP]
> For now we are creating plain indexes. In a production setup we would create an **index template** first — defining the mappings, the number of shards/replicas, and an ILM policy for rollover and retention — and let the first ingest auto-create the index from the template. Part 6 (traces) will introduce a template; the sister series *Elastic Stack for MuleSoft Observability* covers the ECS template for logs in detail.

---

### Step 2 — Create the `mule-logs-writer` and `mule-traces-writer` Roles

We will create one role per index, each granting `read` and `write` only — no cluster-level privileges, no cross-index access. The privilege set also includes `auto_configure` so the writer can let Elasticsearch create the index on first write if we ever delete and recreate it.

#### Option A — Kibana UI

For each role:

1. From the Kibana home page, go to **Stack Management → Security → Roles**, then **Create role**.
2. For `mule-logs-writer`:
   - **Role name:** `mule-logs-writer`
   - **Description:** `Write access to the mule-logs index for Mule applications`
   - **Index privileges → Indices:** `mule-logs`
   - **Privileges:** `read`, `write`, `auto_configure`
3. Click **Create role**.
4. Repeat with **role name** `mule-traces-writer`, **indices** `mule-traces`, same privilege set.

![](Pasted%20image%2020260608115841.png)

#### Option B — Elasticsearch REST API

```bash
# mule-logs-writer
curl -X POST -u elastic:$ELASTIC_PASSWORD \
  "http://10.0.1.20:9200/_security/role/mule-logs-writer" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": [
      {
        "names": ["mule-logs"],
        "privileges": ["read", "write", "auto_configure"]
      }
    ]
  }'

# mule-traces-writer
curl -X POST -u elastic:$ELASTIC_PASSWORD \
  "http://10.0.1.20:9200/_security/role/mule-traces-writer" \
  -H "Content-Type: application/json" \
  -d '{
    "indices": [
      {
        "names": ["mule-traces"],
        "privileges": ["read", "write", "auto_configure"]
      }
    ]
  }'
```

We expect for each:

```json
{ "role": { "created": true } }
```

> [!IMPORTANT]
> Resist the temptation to grant `all` or `manage` "just in case." If we ever need to add a privilege we update the role — but starting tight forces us to discover any over-broad assumption early, when it is cheap to fix. The most common reason to relax this is **index templates**: if a role needs to create indexes via a template at first write, `auto_configure` covers it; if it needs to manage the templates themselves, that is a different (cluster-level) role and should not live here.

---

### Step 3 — Create the `mule-logger` and `mule-tracer` Users

We will create two service users, one bound to each role. These are the credentials our Mule apps and the OTel Collector will use.

#### Option A — Kibana UI

For each user:

1. From **Stack Management → Security → Users**, click **Create user**.
2. For `mule-logger`:
   - **Username:** `mule-logger`
   - **Password:** a strong password — we will save this in our secret manager
   - **Full name:** `Mule Logger Service Account`
   - **Email:** `mule-logger@example.com`
   - **Roles:** `mule-logs-writer`
3. Click **Create user**.
4. Repeat for `mule-tracer` with role `mule-traces-writer`.

![](Pasted%20image%2020260608120101.png)

#### Option B — Elasticsearch REST API

```bash
# mule-logger
curl -X POST -u elastic:$ELASTIC_PASSWORD \
  "http://10.0.1.20:9200/_security/user/mule-logger" \
  -H "Content-Type: application/json" \
  -d '{
    "password": "<MULE_LOGGER_PASSWORD>",
    "roles": ["mule-logs-writer"],
    "full_name": "Mule Logger Service Account",
    "email": "mule-logger@example.com"
  }'

# mule-tracer
curl -X POST -u elastic:$ELASTIC_PASSWORD \
  "http://10.0.1.20:9200/_security/user/mule-tracer" \
  -H "Content-Type: application/json" \
  -d '{
    "password": "<MULE_TRACER_PASSWORD>",
    "roles": ["mule-traces-writer"],
    "full_name": "Mule Tracer Service Account",
    "email": "mule-tracer@example.com"
  }'
```

We expect for each:

```json
{ "created": true }
```

For the rest of this tutorial we will export the passwords to environment variables so the verification commands stay readable:

```bash
export MULE_LOGGER_PASSWORD='<MULE_LOGGER_PASSWORD>'
export MULE_TRACER_PASSWORD='<MULE_TRACER_PASSWORD>'
```

> [!WARNING]
> Avoid putting service-account passwords in shell history or in role descriptions. In production, generate them with `openssl rand -base64 24`, store them in a secret manager (AWS Secrets Manager, HashiCorp Vault, Anypoint Secure Properties), and rotate them on a schedule.
>
> On HTTP-only clusters this matters even more — the password is base64-encoded in the `Authorization` header, **not encrypted**. Anyone tcpdumping the VPC subnet can read it.

---

### Step 4 — Test Each User Can Write to Its Index

We will use Postman (or `curl`) to write one document with `mule-logger` to `mule-logs` and one with `mule-tracer` to `mule-traces`. If both roles are wired correctly, both writes succeed.

In Postman, configure the first request:

| Field   | Value                                                                                                                |
| ------- | -------------------------------------------------------------------------------------------------------------------- |
| Method  | `POST`                                                                                                               |
| URL     | `http://10.0.1.20:9200/mule-logs/_doc`                                                                               |
| Auth    | Basic Auth — username `mule-logger`, password `$MULE_LOGGER_PASSWORD`                                                |
| Headers | `Content-Type: application/json`                                                                                     |
| Body    | `{ "@timestamp": "2026-06-08T10:00:00Z", "service.name": "hello-app", "log.level": "INFO", "message": "It works!" }` |

We expect a `201 Created` response with the document metadata:

```json
{
  "_index": "mule-logs",
  "_id": "qB2C...",
  "_version": 1,
  "result": "created",
  "_shards": { "total": 2, "successful": 1, "failed": 0 },
  "_seq_no": 0,
  "_primary_term": 1
}
```

> [!TIP]
> We are sending **ECS-style** field names (`@timestamp`, `service.name`, `log.level`) on purpose. Kibana's prebuilt visualizations and the unified dashboard in Part 7 know how to read them. The full ECS template is covered in the sister series.

![](Pasted%20image%2020260608120456.png)

The same call from `curl` — note no `-k` and no `https://`, since the cluster is HTTP-only:

```bash
curl -X POST -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_doc" \
  -H "Content-Type: application/json" \
  -d '{ "@timestamp": "2026-06-08T10:00:00Z", "service.name": "hello-app", "log.level": "INFO", "message": "It works!" }'
```

Now the second request, swapping user, index, and payload:

```bash
curl -X POST -u mule-tracer:$MULE_TRACER_PASSWORD \
  "http://10.0.1.20:9200/mule-traces/_doc" \
  -H "Content-Type: application/json" \
  -d '{
    "@timestamp": "2026-06-08T10:00:00Z",
    "service.name": "hello-app",
    "trace.id": "0af7651916cd43dd8448eb211c80319c",
    "span.id": "b7ad6b7169203331",
    "span.name": "GET /hello",
    "span.duration_ms": 12
  }'
```

![](Pasted%20image%2020260608120705.png)

---

### Step 5 — Test Authentication Fails with Bad Credentials

A passing test only proves the happy path. We will also confirm the negative path: with the wrong password, Elasticsearch rejects the request.

Change the password in Postman to anything wrong and re-send the log request. We expect a `401 Unauthorized`:

```json
{
  "error": {
    "root_cause": [{
      "type": "security_exception",
      "reason": "unable to authenticate user [mule-logger] for REST request [/mule-logs/_doc]"
    }],
    "status": 401
  }
}
```

If the request still succeeds, our role and user are not enforcing — most likely Elasticsearch security is disabled or the `elasticsearch.yml` was edited away from the Part 1 baseline. Check `xpack.security.enabled: true` in `/etc/elasticsearch/elasticsearch.yml` before continuing.

> [!TIP]
> The same `401` should appear if we change the username instead of the password. This is the kind of test we want in our Mule app's startup health check too — fail fast on bad credentials, before the appender's queue fills up.

---

### Step 6 — Test Cross-Concern Access Is Denied

This is the test that justifies splitting users in the first place. With the **correct** `mule-logger` password, we will try to write into `mule-traces` and confirm Elasticsearch refuses.

```bash
curl -X POST -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-traces/_doc" \
  -H "Content-Type: application/json" \
  -d '{ "should": "fail" }'
```

We expect a `403 Forbidden`:

```json
{
  "error": {
    "type": "security_exception",
    "reason": "action [indices:data/write/index] is unauthorized for user [mule-logger] with effective roles [mule-logs-writer] on indices [mule-traces], this action is granted by the index privileges [create_doc,create,index,write,all]"
  },
  "status": 403
}
```

The mirror test — `mule-tracer` writing to `mule-logs` — should also return `403`. If either succeeds, our role index pattern is wrong; revisit Step 2.

---

### Step 7 — Test Each User Can Read Its Index Back

Restore the correct passwords and run a search with each user against its own index.

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_search?pretty"

curl -u mule-tracer:$MULE_TRACER_PASSWORD \
  "http://10.0.1.20:9200/mule-traces/_search?pretty"
```

We expect each call to return the one document we indexed in Step 4. If a user can write but cannot read, we forgot the `read` privilege when we created the role — go back to Step 2.

![](Pasted%20image%2020260608120829.png)

---

### Step 8 — Create the Kibana Data Views

Documents in Elasticsearch are not visible to Kibana's Discover, Dashboards, or Lens UIs until we wrap them in a **data view** (formerly "index pattern"). We will create one for each signal.

1. In Kibana, click the menu button on the top left and go to **Stack Management → Data Views**, then **Create data view**.
2. For the logs view:
   - **Name:** `Mule Logs`
   - **Index pattern:** `mule-logs*` (the trailing `*` future-proofs us for `mule-logs-2026.06.08` and similar rollover names)
   - **Timestamp field:** `@timestamp` — we wrote this field in Step 4, so it appears in the dropdown.
3. Click **Save data view to Kibana**.
4. Repeat with **Name** `Mule Traces`, **Index pattern** `mule-traces*`, **Timestamp field** `@timestamp`.

> [!TIP]
> Picking `@timestamp` here is what unlocks the Discover time filter and any time-based visualization we build in Part 7. If we ever forget it, we can edit the data view later from Stack Management → Data Views.

![](Pasted%20image%2020260608121414.png)


---

### Step 9 — Browse the Documents in Discover

With both data views in place we can finally see our documents.

1. In Kibana, go to **Analytics → Discover**.
2. Pick the `Mule Logs` data view from the dropdown at the top left. Set the time filter to **Last 24 hours**.
3. We should see one row, with the `message` field set to `It works!` and `service.name` set to `hello-app`.
4. Switch the data view to `Mule Traces`. We should see one row with the `trace.id`, `span.id`, and `span.duration_ms` fields populated.

![](Pasted%20image%2020260608121555.png)

![](Pasted%20image%2020260608121609.png)
If Discover shows the data view but no rows, the time filter at the top right is the most common culprit — widen it to **Last 24 hours** or longer.

---

## Verification

We will run four checks to confirm everything is wired up correctly.

**1. Both indexes exist and each has at least one document.**

```bash
curl -u elastic:$ELASTIC_PASSWORD \
  "http://10.0.1.20:9200/mule-logs,mule-traces/_count?pretty"
```

Expect `"count": 2` (one document per index).

**2. Each service user has the right role.**

```bash
curl -u elastic:$ELASTIC_PASSWORD \
  "http://10.0.1.20:9200/_security/user/mule-logger,mule-tracer?pretty"
```

Expect `mule-logger.roles == ["mule-logs-writer"]` and `mule-tracer.roles == ["mule-traces-writer"]`.

**3. Each user is denied the other concern.**

```bash
curl -X POST -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/mule-traces/_doc" \
  -H "Content-Type: application/json" -d '{}'
# Expect 403

curl -X POST -u mule-tracer:$MULE_TRACER_PASSWORD \
  "http://10.0.1.20:9200/mule-logs/_doc" \
  -H "Content-Type: application/json" -d '{}'
# Expect 403
```

**4. Neither user can read system indices.**

```bash
curl -u mule-logger:$MULE_LOGGER_PASSWORD \
  "http://10.0.1.20:9200/.security/_search?pretty"
# Expect 403

curl -u mule-tracer:$MULE_TRACER_PASSWORD \
  "http://10.0.1.20:9200/.security/_search?pretty"
# Expect 403
```

If all four checks pass, we are done.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `PUT /mule-logs` returns `400 resource_already_exists_exception` | Index was created earlier (e.g. by an earlier test ingest auto-creating it) | Delete and recreate: `curl -X DELETE -u elastic:$ELASTIC_PASSWORD http://10.0.1.20:9200/mule-logs` |
| `mule-logger` write returns `403 Forbidden` on `mule-logs` | Role was created without `write` (or `auto_configure` if the index does not exist yet) | `GET /_security/role/mule-logs-writer`, add the missing privilege |
| `mule-logger` write to `mule-traces` returns `201 Created` (not `403`) | Role index pattern is too broad (e.g. `mule-*`) or role was assigned both writers | Re-check Step 2; the `names` array must list exactly one index |
| `curl http://10.0.1.20:9200` returns `received plaintext http traffic on an https channel` | `xpack.security.http.ssl.enabled` was not flipped to `false` | Re-check Part 1 Step 6, restart the service |
| Bad credentials still succeed | `xpack.security.enabled: false` in `elasticsearch.yml` | Set it back to `true` and restart the service |
| Data view created but Discover shows no documents | Time filter excludes them | Widen the time filter, or check that the document's `@timestamp` is recent |
| `curl ... /_count` from an HTTP client returns `401 missing authentication credentials` | Forgot the `-u user:password` flag | Add it; every call needs Basic Auth |
| Postman times out from the workstation | Reaching `10.0.1.20:9200` from outside the VPC will not work | Run the requests from a host inside the VPC, or set up an SSH tunnel to the ES host |

---

## What We Covered

- We created **two** dedicated indexes — `mule-logs` and `mule-traces` — both via the Kibana UI and via the Elasticsearch REST API.
- We created **two** least-privilege roles, `mule-logs-writer` and `mule-traces-writer`, each scoped to a single index with `read`, `write`, and `auto_configure`.
- We created **two** service users, `mule-logger` and `mule-tracer`, each bound to exactly one role.
- We tested all four pieces end-to-end: a write per user, a deliberate auth failure, a deliberate cross-concern denial, and a read per user.
- We created Kibana data views over both indexes and confirmed our documents are visible in Discover, with `@timestamp` driving the time filter.

The cluster is now ready to receive Mule data with one credential per concern, leaving the `elastic` superuser out of every Mule app and every OTel Collector config from this point forward.

> ➡️ **Next up:** In Part 4 we will build the first hello-world Mule app on Mule 4.11 and wire its `log4j2.xml` `HttpAppender` to the `mule-logs` index using the `mule-logger` credentials we just created — over plain HTTP, no truststore needed.

---

## References

- [Indices APIs — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices.html)
- [Built-in roles & privileges — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-roles.html)
- [Create or update role API — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-put-role.html)
- [Create or update users API — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-put-user.html)
- [Index privileges (`auto_configure`, `write`, `read`) — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-privileges.html#privileges-list-indices)
- [Create a data view — Kibana Docs](https://www.elastic.co/guide/en/kibana/current/data-views.html)
- [Discover — Kibana Docs](https://www.elastic.co/guide/en/kibana/current/discover.html)
- [Elastic Common Schema (ECS) — Elastic Docs](https://www.elastic.co/guide/en/ecs/current/index.html)
