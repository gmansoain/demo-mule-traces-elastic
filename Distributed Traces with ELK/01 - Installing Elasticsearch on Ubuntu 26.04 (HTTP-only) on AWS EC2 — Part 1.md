---
title: Installing Elasticsearch on Ubuntu 26.04 (HTTP-only) on AWS EC2 — Part 1
slug: install-elasticsearch-ubuntu-26-04-http-aws-ec2
description: Install Elasticsearch 8.x on Ubuntu Server 26.04 LTS hosted on AWS EC2, with security enabled but HTTP TLS turned off — a clean demo backend for Mule logs and traces.
author: Gonzalo Marcos
date: 2026-06-08
status: not validated
lang: en
category: observability
tags:
  - elasticsearch
  - ubuntu
  - linux
  - aws
  - installation
  - logging
  - english
  - series
series: Sending Logs and Distributed Traces to Elastic
series_part: 1
type: tutorial
difficulty: intermediate
read_time: 14
platform:
  - aws
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![AWS](https://img.shields.io/badge/AWS-FF9900?logo=amazonaws&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 1](https://img.shields.io/badge/Part-1-16a085) ![14 min](https://img.shields.io/badge/Read_Time-14_min-lightgrey)

# Installing Elasticsearch on Ubuntu 26.04 (HTTP-only) on AWS EC2 — Part 1

Elasticsearch is the storage and search engine at the core of the Elastic Stack. For Mule architects and developers, it is the natural home for application logs and distributed traces. Once we have data flowing into it, troubleshooting a misbehaving Mule app stops being a hunt through CloudHub log files and becomes a Kibana query.

In this tutorial we will install Elasticsearch 8.x on a single Ubuntu Server **26.04 LTS** host running on **AWS EC2**, using the official Elastic APT repository. We will keep `xpack.security` **enabled** (so Basic Auth still applies) and we will only disable **HTTP TLS** so that downstream Mule log4j2 appenders can talk to the cluster without truststore plumbing. Transport TLS and enrollment stay **on** — the node will not start without them. The trade-off is explicit and demo-only; we will call it out clearly and link to the production-grade variant.

By the end of the post we will have a healthy Elasticsearch node, a known password for the built-in `elastic` user, and a working HTTP API ready for Kibana (Part 2) to connect to.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Next:** Part 2 — Installing Kibana on Ubuntu 26.04 on a separate EC2 instance

> [!WARNING]
> **HTTP-only, demo-grade.** This series intentionally exposes Elasticsearch on **plain HTTP**. Authentication still works (Basic Auth over the `elastic` user and least-privilege service users), but credentials and payloads travel **unencrypted** on the network. This is acceptable for a self-contained AWS lab inside a private VPC and a tight security group; it is **not** appropriate for any environment where the data or the network is shared. For the production-grade variant with TLS, enrollment tokens, and certificate rotation, see the sister series *Elastic Stack for MuleSoft Observability*.

---

## What We Will Cover

- Provision an Ubuntu Server 26.04 LTS EC2 instance and lock down its security group.
- Add the Elastic 8.x APT repository and its signing key.
- Install Elasticsearch and capture (or reset) the auto-generated `elastic` superuser password.
- Disable **HTTP** TLS in `elasticsearch.yml` while keeping authentication, transport TLS, and enrollment on.
- Tune the JVM heap to match the host.
- Start the service, run smoke tests over plain HTTP, and verify cluster health.

---

## Prerequisites

Before we start, we will need:

- An **AWS account** with permission to launch EC2 instances and edit security groups in a private VPC.
- An **Ubuntu Server 26.04 LTS** EC2 instance (these steps also work on 24.04 LTS without changes).
- A user with `sudo` privileges on that instance.
- `curl`, `wget`, and `gpg` installed (all present on the Ubuntu Server cloud image).
- Outbound HTTPS access to `artifacts.elastic.co`.
- Port `9200/tcp` reachable from the future Kibana host, the Mule egress IPs, and the OTel Collector — and from nowhere else.

We will confirm the Ubuntu version before doing anything else:

```bash
lsb_release -a
```

> [!NOTE]
> *Screenshot — terminal output of `lsb_release -a` confirming Ubuntu 26.04 LTS.*

> [!IMPORTANT]
> **Ubuntu 26.04 + Elastic apt repo.** The Elastic 8.x apt source uses `stable main` and is not pinned to an Ubuntu codename, so the same source list works on 24.04 and 26.04. If `apt-get update` complains about the codename or fails on the GPG key, jump to the **Troubleshooting** table at the end of this post.

### Step 0 — Set the hostname (do this before installing)

A fresh AWS EC2 Ubuntu image comes up with a machine-generated name like `ip-172-31-13-241.eu-central-1.compute.internal`. We will rename it **before** installing Elasticsearch so the auto-generated config picks up a name we actually want to read in logs and dashboards.

We will set the static hostname:

```bash
sudo hostnamectl set-hostname mule-elk-01
```

We will map the new name to loopback so utilities like `sudo` and `mail` do not warn about an unresolvable hostname:

```bash
echo "127.0.1.1 mule-elk-01" | sudo tee -a /etc/hosts
```

> [!WARNING]
> On Ubuntu EC2 images, `cloud-init` rewrites the hostname on every boot by default. Without the next step, our reboot will silently flip the hostname back to `ip-172-31-...`.

We will tell `cloud-init` to keep our hostname:

```bash
sudo sed -i 's/^preserve_hostname:.*$/preserve_hostname: true/' /etc/cloud/cloud.cfg
```

We will log out and back in so the shell prompt picks up the change, then confirm:

```bash
hostnamectl
```

We expect `Static hostname: mule-elk-01` in the output.

> [!NOTE]
> *Screenshot — `hostnamectl` output with `Static hostname: mule-elk-01`.*

### Hardware sizing

Elastic publishes minimum requirements that are fine for a lab and tight for production. The table below is a useful starting point for the Elasticsearch host. For a single-node lab on AWS, **t3.medium** is the floor; **t3.large** or **m6i.large** is more comfortable.

| Component | Minimum (lab) | Recommended (single-node demo) |
| --- | --- | --- |
| EC2 instance type | `t3.medium` (2 vCPU / 4 GB) | `t3.large` (2 vCPU / 8 GB) or `m6i.large` |
| JVM heap | 2 GB | 4 GB |
| EBS root volume | 20 GB gp3 | 50 GB gp3 |
| Network | default | default (private subnet) |

> [!TIP]
> Elasticsearch is happiest on SSDs. AWS gp3 EBS is a fine starting point; gp2 also works but reach for gp3 if you have a choice — the IOPS baseline is higher.

### Security group

The instance's security group should allow inbound:

- `22/tcp` from our admin IP (SSH).
- `9200/tcp` from the **Kibana SG**, the **OTel Collector SG**, and the Mule egress IPs (we will add these as we stand each component up; for Part 1 just our admin IP is enough).

Outbound: leave the default `0.0.0.0/0` so the apt update can reach `artifacts.elastic.co`.

---

## Overview

We will run a single-node Elasticsearch deployment on the Ubuntu host, with security enabled but HTTP TLS turned off. The topology for this post is just the box itself — Kibana, the Mule apps, and the OTel Collector arrive in later parts.

```
┌──────────────────────────────────────────────┐
│  AWS EC2 — Ubuntu Server 26.04 LTS           │
│                                              │
│              ┌──────────────┐                │
│              │ Elasticsearch│                │
│              │  :9200 (HTTP)│                │
│              │  Basic Auth  │                │
│              └──────────────┘                │
│                     ▲                        │
└─────────────────────┼────────────────────────┘
                      │ HTTP + Basic Auth
                      │ (private VPC only)
              Kibana / Mule / OTel
```

---

## Table of Contents

1. [Step 1 — Add the Elastic Signing Key](#step-1--add-the-elastic-signing-key)
2. [Step 2 — Install `apt-transport-https`](#step-2--install-apt-transport-https)
3. [Step 3 — Register the Elastic 8.x Repository](#step-3--register-the-elastic-8x-repository)
4. [Step 4 — Install Elasticsearch](#step-4--install-elasticsearch)
5. [Step 5 — Capture or Reset the `elastic` Password](#step-5--capture-or-reset-the-elastic-password)
6. [Step 6 — Configure Elasticsearch (HTTP-only)](#step-6--configure-elasticsearch-http-only)
7. [Step 7 — Tune the JVM Heap](#step-7--tune-the-jvm-heap)
8. [Step 8 — Start and Enable the Service](#step-8--start-and-enable-the-service)
9. [Step 9 — Smoke Test the HTTP API](#step-9--smoke-test-the-http-api)
10. [Verification](#verification)
11. [Troubleshooting](#troubleshooting)

---

### Step 1 — Add the Elastic Signing Key

APT will refuse to install packages from a repository it does not trust. We will download the Elastic GPG key and store it in the keyring directory `apt` reads from.

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg
```

The `--dearmor` flag converts the ASCII-armored key into the binary format `apt` expects.

---

### Step 2 — Install `apt-transport-https`

The Elastic repository is served over HTTPS. On a fresh Ubuntu Server install, `apt` cannot fetch HTTPS repositories until we install the helper package.

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https
```

If the package is already installed, `apt` will report it and exit cleanly.

---

### Step 3 — Register the Elastic 8.x Repository

We will add the Elastic 8.x APT source list and bind it to the keyring we just installed. The `signed-by` directive is what links the two — without it, `apt` will not trust the repository.

```bash
echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
```

We will refresh the package index so `apt` picks up the new source:

```bash
sudo apt-get update
```

> [!IMPORTANT]
> Pin the major version in the source list (`8.x` here). Following `latest` means an `apt upgrade` could pull a new major version and break our cluster. We want major upgrades to be a deliberate decision, not a side effect of `unattended-upgrades`.

---

### Step 4 — Install Elasticsearch

We will pin the version explicitly. At the time of writing, the current 8.x release is `8.15.3` — we will use that. The pattern is `apt-get install elasticsearch=<VERSION>`:

```bash
sudo apt-get install elasticsearch=8.15.3
```

The install takes a minute. APT prints two important things during the process:

1. The auto-generated password for the `elastic` superuser.
2. The path to the auto-generated security configuration.

We will read the next step before the terminal scrolls past them.

> [!NOTE]
> *Screenshot — `apt-get install elasticsearch=8.15.3` console output, including the security autoconfiguration block with the generated `elastic` password.*

---

### Step 5 — Capture or Reset the `elastic` Password

When Elasticsearch installs, it generates a password for the built-in `elastic` superuser and prints it to the install output. **This is the only time it appears in plain text.** We will copy it into a password manager before doing anything else.

The output looks like this:

```text
--------------------------- Security autoconfiguration information ------------------------------

Authentication and authorization are enabled.
TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is : <PASSWORD_HERE>
```

> [!TIP]
> **If we missed it**, or if we want a known password for this lab, we can reset it on demand. Elasticsearch only stores the hash, so the original is unrecoverable, but the reset CLI is one command:
>
> ```bash
> sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -i
> ```
>
> The `-i` flag prompts for the new password interactively. Drop it for a randomly generated one printed once to stdout, or use `-a` (auto) for scripting.

For convenience in the rest of this tutorial we will export the password to an environment variable. We will replace the placeholder with our actual password:

```bash
export ELASTIC_PASSWORD='<PASSWORD_HERE>'
```

> [!WARNING]
> **HTTP-only means this password travels in clear text.** Once we open `:9200` to anything beyond `localhost`, anyone on the same network segment can sniff it. Keep the security group tight (private subnet, specific source SGs) until we add a TLS-terminating proxy in a later post.

---

### Step 6 — Configure Elasticsearch (HTTP-only)

The package installs a working baseline configuration at `/etc/elasticsearch/elasticsearch.yml`, including an auto-generated `xpack.security.*` block with HTTP and transport TLS enabled. For this series we will keep `xpack.security.enabled` **on** so the Basic Auth model still applies, and turn HTTP and transport TLS **off** so log4j2 appenders and the OTel Collector can talk to the cluster without truststore plumbing.

We will open the file:

```bash
sudo nano /etc/elasticsearch/elasticsearch.yml
```

We will set or confirm the following values. In the auto-generated security block at the bottom of the same file we will leave `xpack.security.enabled: true`, leave `xpack.security.enrollment.enabled` and `xpack.security.transport.ssl` exactly as the installer wrote them, and **only** flip `xpack.security.http.ssl.enabled` to `false`.

```yaml
# /etc/elasticsearch/elasticsearch.yml

# A descriptive name helps when this node joins a real cluster later.
cluster.name: mule-elk
node.name: mule-elk-01

# Default data and log paths — leave as-is unless we have a separate data volume.
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch

# Bind to the private interface so Kibana/Mule/OTel can reach it.
# Replace 10.0.1.20 with the private IP of this EC2 instance.
network.host: 10.0.1.20
http.port: 9200

# Single-node cluster — skips master-election bootstrap checks.
discovery.type: single-node

# ----------------------- Security (HTTP-only profile) ----------------------
# Authentication stays on — Basic Auth applies to every request.
xpack.security.enabled: true

# HTTP layer: plain HTTP, no TLS — this is the ONLY TLS flag we touch.
xpack.security.http.ssl:
  enabled: false

# IMPORTANT: leave `xpack.security.enrollment.enabled` and the
# `xpack.security.transport.ssl.*` block exactly as the installer wrote them.
# Setting either to `false` makes the node fail to start.
```

Save and close (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

> [!WARNING]
> **Do not disable transport TLS or enrollment.** The installer writes `xpack.security.enrollment.enabled: true` and a `xpack.security.transport.ssl` block (`enabled: true`, `keystore.path: certs/transport.p12`, ...). Setting either to `false` causes the node to refuse to start with errors like `Transport SSL must be enabled if security is enabled`. Transport TLS only secures intra-cluster traffic between nodes — it does **not** affect our HTTP appender. We leave it on.

> [!IMPORTANT]
> The auto-config also writes `xpack.security.http.ssl.keystore.path: certs/http.p12` near the bottom. With `xpack.security.http.ssl.enabled: false` that line is inert; we can leave it or comment it out. We will leave it so a future "flip TLS back on" is just one boolean change.

> [!NOTE]
> **Why bind to a private IP instead of `0.0.0.0`?**
> The moment `network.host` is anything other than a loopback address, Elasticsearch flips into [production mode](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html) and enforces strict bootstrap checks (`vm.max_map_count >= 262144`, file descriptor limits, memory locking). A specific private IP is preferred over `0.0.0.0` for the same reason a security group is preferred over `0.0.0.0/0` — limit the surface to the network we trust.

> [!TIP]
> Pre-tune the kernel before the first start so the bootstrap check passes:
>
> ```bash
> sudo sysctl -w vm.max_map_count=262144
> echo 'vm.max_map_count=262144' | sudo tee /etc/sysctl.d/99-elasticsearch.conf
> ```

---

### Step 7 — Tune the JVM Heap

Elasticsearch is a JVM process, and the JVM heap is the single most important setting for performance and stability. The defaults aim for 50% of system RAM, capped at ~31 GB. On small hosts the default works; on bigger hosts we will set it explicitly.

We will create a heap override file. Custom files in `/etc/elasticsearch/jvm.options.d/` take precedence over the defaults and survive package upgrades.

```bash
sudo nano /etc/elasticsearch/jvm.options.d/heap.options
```

For an 8 GB host (`t3.large` / `m6i.large`) we will allocate 4 GB to the heap (50% of system RAM, leaving the rest for the OS page cache, which Lucene relies on heavily):

```ini
-Xms4g
-Xmx4g
```

For a 4 GB host, use `-Xms2g` / `-Xmx2g`. The two values must always match — Elasticsearch will refuse to start otherwise.

Save and close.

> [!WARNING]
> Never set the heap above ~31 GB. The JVM uses compressed object pointers up to that boundary; cross it and pointer sizes double, wasting memory and hurting performance. If our host has 64 GB of RAM, the heap should still be ~30 GB and the rest goes to the page cache.

---

### Step 8 — Start and Enable the Service

The installer registers `elasticsearch.service` with systemd but does not start it. We will reload systemd, enable the service for boot, and start it now.

```bash
sudo systemctl daemon-reload
sudo systemctl enable elasticsearch.service
sudo systemctl start elasticsearch.service
```

We will check the service is running:

```bash
sudo systemctl status elasticsearch.service
```

We will look for `active (running)` in the output. Elasticsearch can take 30–60 seconds to become responsive on the first start while it initializes the indices.

If anything looks off, we will tail the journal:

```bash
sudo journalctl -u elasticsearch.service -f
```

> [!NOTE]
> *Screenshot — `systemctl status elasticsearch.service` showing the service in `active (running)` state.*

---

### Step 9 — Smoke Test the HTTP API

We will hit the root endpoint to confirm Elasticsearch responds. Because HTTP TLS is off, we drop the `-k` and the `https://` from every command — plain HTTP plus Basic Auth:

```bash
curl -u elastic:$ELASTIC_PASSWORD http://localhost:9200
```

We expect a JSON response that looks like this:

```json
{
  "name" : "mule-elk-01",
  "cluster_name" : "mule-elk",
  "cluster_uuid" : "...",
  "version" : {
    "number" : "8.15.3",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "...",
    "build_date" : "...",
    "build_snapshot" : false,
    "lucene_version" : "9.11.1",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

The `cluster_name` and `name` values should match what we set in Step 6. If they do, our configuration is loaded.

> [!NOTE]
> *Screenshot — `curl http://localhost:9200` over Basic Auth, returning the JSON cluster banner.*

We will also verify the call from the **private IP** (the one we bound to in Step 6) — this confirms the security group and `network.host` are wired up correctly:

```bash
curl -u elastic:$ELASTIC_PASSWORD http://10.0.1.20:9200
```

---

## Verification

We will run three checks to confirm Elasticsearch is healthy and ready for Part 2.

**1. The service is running and bound to the expected port.**

```bash
sudo ss -tlnp | grep 9200
```

We expect to see Elasticsearch listening on `10.0.1.20:9200` (the private IP we set), **not** `127.0.0.1:9200`.

**2. The cluster reports a `green` or `yellow` health status.**

```bash
curl -u elastic:$ELASTIC_PASSWORD \
  http://localhost:9200/_cluster/health?pretty
```

We expect a response like:

```json
{
  "cluster_name" : "mule-elk",
  "status" : "green",
  "timed_out" : false,
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1,
  "active_primary_shards" : 0,
  "active_shards" : 0
}
```

A fresh single-node cluster with no indices reports `green`. Once we add data, it will likely flip to `yellow` because replica shards have nowhere to go on a one-node cluster — that is expected and not a problem. `red` means a primary shard failed to allocate and we have a real issue.

**3. We can write and read a document.**

We will create a tiny document, read it back, and delete it. This proves the indexing pipeline is healthy end to end.

```bash
# Index a document
curl -u elastic:$ELASTIC_PASSWORD \
  -X POST "http://localhost:9200/smoke-test/_doc?pretty" \
  -H 'Content-Type: application/json' \
  -d '{"app": "mule-orders", "level": "INFO", "message": "hello elastic"}'

# Read all documents in the index
curl -u elastic:$ELASTIC_PASSWORD \
  "http://localhost:9200/smoke-test/_search?pretty"

# Clean up
curl -u elastic:$ELASTIC_PASSWORD \
  -X DELETE "http://localhost:9200/smoke-test"
```

The first call returns a `_id` and `"result": "created"`. The second returns the document we just wrote. The third returns `"acknowledged": true`. If all three succeed, Elasticsearch is fully operational.

> [!NOTE]
> *Screenshot — three-call sequence (index, search, delete) with their JSON responses.*

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `apt-get update` fails with `NO_PUBKEY` | Signing key not installed or saved to the wrong path | Re-run Step 1 and confirm `/usr/share/keyrings/elasticsearch-keyring.gpg` exists |
| `apt-get update` warns about codename on Ubuntu 26.04 | Local apt config tries to inject a codename into the Elastic repo | The Elastic source uses `stable main`, no codename — confirm the line in `/etc/apt/sources.list.d/elastic-8.x.list` matches Step 3 exactly |
| Elasticsearch service exits immediately after start | OOM: JVM heap too large for the host | Lower `-Xms`/`-Xmx` in `/etc/elasticsearch/jvm.options.d/heap.options`, or upsize the EC2 instance |
| Service starts but `curl http://localhost:9200` hangs | Node still bootstrapping | Wait 60s on first start, then retry; tail `journalctl -u elasticsearch -f` for the "started" log line |
| `curl http://localhost:9200` returns `received plaintext http traffic on an https channel` | The `xpack.security.http.ssl` block was not flipped to `enabled: false` | Re-edit `/etc/elasticsearch/yml`, set `xpack.security.http.ssl.enabled: false`, restart the service |
| `curl` gets `401 missing authentication credentials` | `xpack.security.enabled: true` is doing its job — every call needs Basic Auth | Pass `-u elastic:$ELASTIC_PASSWORD` |
| Lost the bootstrap password from Step 5 | Password never copied; Elasticsearch only stores the hash | Reset it with `sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -i` |
| Node hangs at startup with `bootstrap checks failed: max virtual memory areas` | `vm.max_map_count` below 262144 | Run the sysctl tip in Step 6, then restart the service |
| Node hangs at startup with `not enough master nodes` | `discovery.type: single-node` missing | Add the line in Step 6, restart the service |
| Remote `curl http://10.0.1.20:9200` from the Kibana host times out | EC2 security group does not allow `:9200` from the Kibana SG | Add an inbound rule for `9200/tcp` from the Kibana SG (or the Kibana instance's private IP) |

---

## What We Covered

- We provisioned an Ubuntu 26.04 LTS EC2 instance and renamed it before installing.
- We added the Elastic 8.x APT repository, signed-by the official key.
- We installed Elasticsearch `8.15.3` and captured (or reset) the `elastic` superuser password.
- We configured `elasticsearch.yml` for a single-node deployment with `xpack.security` enabled, **only HTTP TLS off**, and transport TLS / enrollment left as the installer wrote them.
- We tuned the JVM heap to 4 GB and started the service under systemd.
- We smoke-tested the HTTP API — Basic Auth over plain HTTP — by indexing, reading, and deleting a document.

We now have a healthy Elasticsearch node listening on `:9200` over HTTP. It is reachable from the private VPC, secured with Basic Auth, and ready to receive Kibana — but without a UI, exploring data is a `curl` exercise. That is what Part 2 fixes.

> ➡️ **Next up:** In Part 2 we will install Kibana on a separate EC2 instance, point it at this Elasticsearch node over plain HTTP, and reset the `kibana_system` built-in user so the two can speak without admin credentials.

---

## References

- [Install Elasticsearch with Debian Package — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/deb.html)
- [Configuring Elasticsearch — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/settings.html)
- [Setting JVM heap size — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/advanced-configuration.html#set-jvm-heap-size)
- [Bootstrap Checks — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html)
- [`elasticsearch-reset-password` — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/reset-password.html)
- [Set up minimal security for Elasticsearch — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-minimal-setup.html)
- [Cluster health API — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html)
- [Ubuntu Server 26.04 LTS Documentation](https://ubuntu.com/server/docs)
