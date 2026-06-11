---
title: Installing Kibana on Ubuntu 26.04 on a Separate AWS EC2 Instance — Part 2
slug: install-kibana-ubuntu-26-04-aws-ec2-http
description: Install Kibana 8.x on a dedicated Ubuntu 26.04 EC2 instance and connect it over plain HTTP to the Elasticsearch node from Part 1, using the kibana_system service user.
author: Gonzalo Marcos
date: 2026-06-08
status: validated
lang: en
category: observability
tags:
  - kibana
  - elasticsearch
  - ubuntu
  - linux
  - aws
  - installation
  - english
  - series
series: Sending Logs and Distributed Traces to Elastic
series_part: 2
type: tutorial
difficulty: intermediate
read_time: 12
platform:
  - aws
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![AWS](https://img.shields.io/badge/AWS-FF9900?logo=amazonaws&logoColor=white) ![Intermediate](https://img.shields.io/badge/Level-Intermediate-f39c12) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 2](https://img.shields.io/badge/Part-2-16a085) ![12 min](https://img.shields.io/badge/Read_Time-12_min-lightgrey)

# Installing Kibana on Ubuntu 26.04 on a Separate AWS EC2 Instance — Part 2

In [Part 1](#) we installed Elasticsearch on an Ubuntu 26.04 EC2 instance with `xpack.security` enabled and only **HTTP TLS** turned off. The cluster is healthy, listening on a private IP at `:9200`, and reachable with Basic Auth — but without a UI, exploring data is a `curl` exercise.

In this tutorial we will install Kibana on a **separate** Ubuntu 26.04 EC2 instance, point it at the Elasticsearch node from Part 1 over plain HTTP, and authenticate with the built-in `kibana_system` service user. The default enrollment-token flow is not available to us — it assumes HTTPS and a CA fingerprint we no longer have — so we will wire Kibana up by hand, which is shorter than it sounds.

By the end of the post we will have a Kibana UI we can log into as the `elastic` superuser, ready for Part 3 where we create the indexes, roles, and service users that our Mule apps will use.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 1 — Installing Elasticsearch on Ubuntu 26.04 (HTTP-only)](#) · **Next:** Part 3 — Initial Elasticsearch Config: Indexes, Roles, and Users for Logs + Traces

> [!WARNING]
> **HTTP-only, demo-grade.** This tutorial connects Kibana to Elasticsearch over **plain HTTP**. Authentication still applies, but `kibana_system` credentials and every search payload travel **unencrypted** between the two EC2 instances. Acceptable inside a private VPC with a tight security group; **not** appropriate for shared networks.

---

## What We Will Cover

- Provision a second Ubuntu Server 26.04 LTS EC2 instance for Kibana.
- Confirm we can reach the Elasticsearch node from Part 1 over its private IP.
- Reset the password of the `kibana_system` built-in user on Elasticsearch.
- Install Kibana from the Elastic 8.x APT repository, version-matched to Part 1.
- Configure `kibana.yml` for an HTTP-only, cross-host deployment.
- Store the `kibana_system` password in the Kibana keystore.
- Start Kibana, log in as `elastic`, and verify the UI talks to Elasticsearch.

---

## Prerequisites

Before we start, we will need:

- A working Elasticsearch node from [Part 1](#) — known private IP (we will call it `10.0.1.20`) and the `elastic` password.
- A **second** Ubuntu Server 26.04 LTS EC2 instance for Kibana. We will call its private IP `10.0.1.21`.
- A user with `sudo` privileges on the Kibana host.
- Outbound HTTPS access to `artifacts.elastic.co` from the Kibana host.
- The Elasticsearch security group updated to allow inbound `9200/tcp` from the Kibana SG (or from `10.0.1.21/32`).
- The Kibana security group allowing inbound `5601/tcp` from our admin IP (or `22/tcp` only, if we plan to access Kibana via SSH local-forward).
- A browser on a machine that can reach the Kibana host (directly or through an SSH tunnel).

We will confirm Elasticsearch is reachable from the Kibana host, over the private IP and plain HTTP:

```bash
# Run this from the Kibana EC2 instance.
curl -u elastic:$ELASTIC_PASSWORD http://10.0.1.20:9200/_cluster/health?pretty
```

We expect a `green` or `yellow` status. Anything else means the cluster, the security group, or the bind address from Part 1 needs another look — installing Kibana on top of an unreachable cluster just adds noise.

![](Installing%20Kibana%20on%20Ubuntu%2026.04%20on%20a%20Separate%20AWS%20EC2%20Instance%20—%20Part%202.png)

### Hardware sizing

| Component | Minimum (lab) | Recommended (single-team demo) |
| --- | --- | --- |
| EC2 instance type | `t3.small` (2 vCPU / 2 GB) | `t3.medium` (2 vCPU / 4 GB) |
| EBS root volume | 10 GB gp3 | 20 GB gp3 |
| Network | default | default (same private subnet as Elasticsearch) |

> [!TIP]
> Keep Kibana in the **same subnet / AZ** as the Elasticsearch node. Kibana hits Elasticsearch on every page render, and a single-AZ link is the cheapest, lowest-latency path.

---

## Overview

We are extending Part 1 with a second EC2 instance running Kibana. Kibana talks to Elasticsearch over plain HTTP using the built-in `kibana_system` service account, and the user reaches Kibana on `:5601`.

```
┌──────────────────────────────────────┐         ┌──────────────────────────────────────┐
│  AWS EC2 — Ubuntu 26.04 LTS          │         │  AWS EC2 — Ubuntu 26.04 LTS          │
│  Elasticsearch host (Part 1)         │         │  Kibana host (this post)             │
│                                      │         │                                      │
│  10.0.1.20                           │ HTTP    │  10.0.1.21                           │
│  ┌──────────────┐                    │◀────────│  ┌──────────────┐                    │
│  │ Elasticsearch│   Basic Auth       │         │  │    Kibana    │                    │
│  │  :9200 (HTTP)│   kibana_system    │         │  │    :5601     │                    │
│  └──────────────┘                    │         │  └──────────────┘                    │
└──────────────────────────────────────┘         └────────────────▲─────────────────────┘
                                                                  │ Browser
                                                              SSH tunnel or direct
```

---

## Table of Contents

1. [Step 1 — Provision the Kibana EC2 Instance](#step-1--provision-the-kibana-ec2-instance)
2. [Step 2 — Add the Elastic Repo on the Kibana Host](#step-2--add-the-elastic-repo-on-the-kibana-host)
3. [Step 3 — Install Kibana](#step-3--install-kibana)
4. [Step 4 — Reset the `kibana_system` Password on Elasticsearch](#step-4--reset-the-kibana_system-password-on-elasticsearch)
5. [Step 5 — Configure `kibana.yml`](#step-5--configure-kibanayml)
6. [Step 6 — Store the `kibana_system` Password in the Keystore](#step-6--store-the-kibana_system-password-in-the-keystore)
7. [Step 7 — Start and Enable the Service](#step-7--start-and-enable-the-service)
8. [Step 8 — Reach the UI and Log In](#step-8--reach-the-ui-and-log-in)
9. [Verification](#verification)
10. [Troubleshooting](#troubleshooting)

---

### Step 1 — Provision the Kibana EC2 Instance

We will launch a second EC2 instance in the **same VPC and subnet** as the Elasticsearch host so the cross-instance traffic stays on the AWS internal network.

- AMI: **Ubuntu Server 26.04 LTS**.
- Instance type: `t3.small` (lab) or `t3.medium`.
- Storage: 10–20 GB gp3.
- Security group: a new SG (`kibana-sg`) allowing inbound `22/tcp` from our admin IP and inbound `5601/tcp` from our admin IP (drop the `5601` rule if we plan to tunnel via SSH).

After the instance is up, we will rename it before installing anything (same reasons as Part 1):

```bash
sudo hostnamectl set-hostname mule-elk-kibana-01
echo "127.0.1.1 mule-elk-kibana-01" | sudo tee -a /etc/hosts
sudo sed -i 's/^preserve_hostname:.*$/preserve_hostname: true/' /etc/cloud/cloud.cfg
```

We will log out and back in, then confirm:

```bash
hostnamectl
```

Then we will go back to the **Elasticsearch security group** and add an inbound rule for `9200/tcp` from `kibana-sg` (preferred) or from `10.0.1.21/32`. Without this rule, Kibana cannot reach the cluster and every error in this post is a dead end.



---

### Step 2 — Add the Elastic Repo on the Kibana Host

The Kibana host is a fresh box — the Elastic 8.x APT repository is not registered yet. We will repeat the same three commands from Part 1.

```bash
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

sudo apt-get update
sudo apt-get install -y apt-transport-https

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt-get update
```

> [!IMPORTANT]
> Pin the major version (`8.x`) here too, for the same reason as in Part 1 — we want major upgrades to be a deliberate decision.

---

### Step 3 — Install Kibana

We will install Kibana pinned to the **same minor version** as the Elasticsearch node from Part 1 (`8.15.3`).

```bash
sudo apt-get install kibana=8.15.3
```

> [!IMPORTANT]
> Elasticsearch and Kibana must run on matching minor versions. Mismatches surface as misleading errors like "Kibana server is not ready yet" that look like network problems but are actually compatibility checks failing. If we upgrade one, we upgrade the other in lockstep.

The installer drops the Kibana files in the standard locations:

| Path                 | Purpose                                      |
| -------------------- | -------------------------------------------- |
| `/etc/kibana/`       | Configuration files (`kibana.yml`, keystore) |
| `/usr/share/kibana/` | Binaries, plugins, Node.js runtime           |
| `/var/lib/kibana/`   | Persistent state                             |
| `/var/log/kibana/`   | Log files                                    |

---

### Step 4 — Reset the `kibana_system` Password on Elasticsearch

Kibana does not log in as the `elastic` superuser on every request — it uses the built-in **`kibana_system`** service user, which has just enough privileges for Kibana to operate. The default enrollment flow generates a random password for it and stores it in the Kibana keystore for us. We are not running enrollment (no HTTPS, no CA fingerprint), so we will set the password by hand.

We will reset it from the **Elasticsearch host** (`10.0.1.20`):

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u kibana_system -i
```

The `-i` flag prompts for the new password interactively. We will pick something strong, store it in our password manager, and copy it — we will paste it into the Kibana keystore in Step 6.

We will sanity-check the new credential, still from the Elasticsearch host:

```bash
curl -u kibana_system:'<KIBANA_SYSTEM_PASSWORD>' http://localhost:9200/_security/_authenticate?pretty
```

The response should include `"username": "kibana_system"` and the role `"kibana_system"`. If we see `401`, the reset did not take effect — re-run the reset command.

> [!IMPORTANT]
> Do **not** put the `elastic` superuser into Kibana's config. The whole point of `kibana_system` is least privilege: if Kibana is compromised, the blast radius is the role assigned to that service user, not the entire cluster.


![](Pasted%20image%2020260608103750.png)
---

### Step 5 — Configure `kibana.yml`

The package ships a working baseline at `/etc/kibana/kibana.yml`. We will edit it to point at our remote Elasticsearch over plain HTTP and to bind Kibana on the right interface.

We will open the file:

```bash
sudo nano /etc/kibana/kibana.yml
```

We will set or confirm:

```yaml
# /etc/kibana/kibana.yml

# --- Server ---
# Bind to 0.0.0.0 only if our security group is tight (admin IP only on :5601).
# If we plan to reach Kibana via SSH local-forward, leave it on 127.0.0.1.
server.host: "0.0.0.0"
server.port: 5601
server.name: "mule-elk-kibana-01"

# Public-facing base URL — fill in with the Kibana host's reachable address.
# Used in alert links, share URLs, and the OIDC redirect (none of which we use yet).
server.publicBaseUrl: "http://10.0.1.21:5601"

# --- Elasticsearch (HTTP, Part 1 cluster) ---
elasticsearch.hosts: ["http://10.0.1.20:9200"]
elasticsearch.username: "kibana_system"
# password lives in the Kibana keystore — see Step 6.
# Do NOT put the password here in plain text.

# --- Logging ---
logging:
  appenders:
    file:
      type: file
      fileName: /var/log/kibana/kibana.log
      layout:
        type: json
  root:
    appenders: [default, file]
    level: info
```

Save and close.

> [!WARNING]
> **Do not put `elasticsearch.password` in `kibana.yml`.** The file is world-readable on most installs (`/etc/kibana/kibana.yml` is `0660` owned by `root:kibana`, but easy to widen by accident). The Kibana keystore in Step 6 is the supported place for credentials.

> [!TIP]
> If we want to keep `:5601` off the public network entirely, leave `server.host: "127.0.0.1"` and reach Kibana through an SSH tunnel:
>
> ```bash
> ssh -L 5601:127.0.0.1:5601 ubuntu@<kibana-host-public-ip>
> ```
>
> We then point the browser at `http://localhost:5601`. This is the lightest production-style setup for a demo cluster.

---

### Step 6 — Store the `kibana_system` Password in the Keystore

Kibana ships a CLI for its keystore. We will create the keystore file (if it does not exist) and add the `kibana_system` password to it.

```bash
sudo /usr/share/kibana/bin/kibana-keystore create
sudo /usr/share/kibana/bin/kibana-keystore add elasticsearch.password
```

The second command prompts us for the value — we paste the password from Step 4 and press `Enter`.

We will verify the key is registered (the value is never printed back):

```bash
sudo /usr/share/kibana/bin/kibana-keystore list
```

We expect a single entry: `elasticsearch.password`.

> [!NOTE]
> The keystore lives at `/etc/kibana/kibana.keystore`. It is owned by `root:kibana` and Kibana reads it at startup. If we ever lose the password, we re-run Step 4 (reset) and Step 6 (re-add) — the keystore does not let us read existing values back.

---

### Step 7 — Start and Enable the Service

```bash
sudo systemctl daemon-reload
sudo systemctl enable kibana.service
sudo systemctl start kibana.service
```

We will check the service is running:

```bash
sudo systemctl status kibana.service
```

Kibana is slower to start than Elasticsearch — give it 30 to 90 seconds on the first run. The Node.js process needs to load all the plugins before it begins listening.

If anything looks off, we will tail the journal:

```bash
sudo journalctl -u kibana.service -f
```

> [!NOTE]
> *Screenshot — `systemctl status kibana.service` showing `active (running)` and the journal tail printing `Kibana is now available`.*

---

### Step 8 — Reach the UI and Log In

If we set `server.host: "0.0.0.0"`, we will open the Kibana host's address in our browser:

```
http://10.0.1.21:5601
```

If we kept `server.host: "127.0.0.1"`, we will start an SSH tunnel from our workstation and open `http://localhost:5601` instead:

```bash
ssh -L 5601:127.0.0.1:5601 ubuntu@<kibana-host-public-ip>
```

Either way, **we should land directly on the Kibana login screen** — there is no enrollment / setup wizard this time. That is the deliberate consequence of skipping the enrollment flow; we wired the cluster connection by hand in Steps 5 and 6.

We will log in as `elastic` with the password from Part 1 (`$ELASTIC_PASSWORD`). The first screen is the Kibana home page, with panels for adding integrations, exploring sample data, and managing the stack.

> [!TIP]
> Click **Try sample data** under "Get started by adding integrations" and load the **Sample web logs** dataset. It indexes a few thousand fake documents into Elasticsearch and gives us a Discover view we can play with — useful for confirming the search and visualization layers work end-to-end before we wire up real Mule logs in Part 3.



---

## Verification

We will run three checks to confirm Kibana is healthy and connected to Elasticsearch.

**1. The service is running and bound to the expected port.**

```bash
sudo ss -tlnp | grep 5601
```

We expect to see Kibana listening on `0.0.0.0:5601` (or `127.0.0.1:5601` if we kept the SSH-tunnel layout).

**2. Kibana's status API reports `available`.**

Kibana exposes its own health endpoint that does not need authentication:

```bash
curl http://localhost:5601/api/status | jq '.status.overall'
```

We expect a JSON object with `"level": "available"`. If we see `"degraded"` or `"critical"`, the response will list which plugin is unhappy and why.

**3. Discover finds the data Elasticsearch already has.**

In the Kibana sidebar we will navigate to **Analytics → Discover**. We will pick the data view created by the sample data load (or any index pattern we have data in) and confirm the document count and the most recent timestamp match what Elasticsearch reports.

If all three checks pass, Kibana is fully connected to Elasticsearch and we are done.

> [!NOTE]
> *Screenshot — Kibana Discover view showing the Sample web logs data view with documents loaded.*

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Kibana log shows "Kibana server is not ready yet" forever | Version mismatch with Elasticsearch | Reinstall Kibana at the exact same version as Elasticsearch (`apt-get install kibana=8.15.3`) |
| Kibana log shows `connect ECONNREFUSED 10.0.1.20:9200` | Security group does not allow `9200/tcp` from the Kibana SG, or Elasticsearch bound to `127.0.0.1` | Add the inbound rule on the Elasticsearch SG; revisit Part 1 Step 6 (`network.host` set to the private IP, not loopback) |
| Kibana log shows `Authentication of [kibana_system] was terminated by realm [reserved]` | Password in the keystore does not match Elasticsearch's `kibana_system` user | Re-run Step 4 (reset on ES) and Step 6 (re-add to keystore), then restart Kibana |
| Kibana log shows `received plaintext http traffic on an https channel` | We pointed Kibana at `https://...` while Elasticsearch is HTTP-only | `elasticsearch.hosts` must be `http://...`, not `https://...` |
| Browser cannot reach Kibana on `:5601` | `kibana-sg` does not allow `5601/tcp` from our admin IP, or `server.host` is `127.0.0.1` | Open the SG rule, or use the SSH-tunnel layout described in Step 8 |
| Login fails with "Invalid credentials" | Wrong `elastic` password | Verify with `curl -u elastic:$ELASTIC_PASSWORD http://10.0.1.20:9200` from the Kibana host |
| Service starts but UI takes >2 minutes to load | First-run plugin optimization is still running | Watch `journalctl -u kibana -f` for "Optimization phase" messages; this only happens once |
| `kibana-keystore list` is empty after `add` | Wrong sudo / wrong user — keystore must be writable by root | Re-run with `sudo`, then `sudo systemctl restart kibana.service` |
| Kibana shows the **enrollment** "Configure Elastic" screen instead of the login | `kibana.yml` is missing `elasticsearch.hosts` and `elasticsearch.username`, so Kibana is in unconfigured mode | Re-check Step 5; restart Kibana |

---

## What We Covered

- We provisioned a second Ubuntu 26.04 EC2 instance in the same VPC as the Elasticsearch node from Part 1.
- We registered the Elastic 8.x apt repo and installed Kibana `8.15.3`, version-matched to the cluster.
- We reset the `kibana_system` built-in user's password on Elasticsearch and stored it in the Kibana keystore.
- We configured `kibana.yml` to point at Elasticsearch over **plain HTTP** with `kibana_system` Basic Auth.
- We started Kibana, logged in as `elastic`, and confirmed Discover can read documents from the cluster.

We now have a working Kibana UI talking to a working Elasticsearch node, both on AWS, both over plain HTTP inside a private VPC. Everything beyond this point is content — indexes, roles, service users for our Mule apps — which is exactly what Part 3 covers.

> ➡️ **Next up:** In Part 3 we will create dedicated indexes for `mule-logs-*` and `mule-traces-*`, define least-privilege roles (`mule-logs-writer`, `mule-traces-writer`), provision the `mule-logger` and `mule-tracer` service users, and add the matching Kibana data views so a single dashboard can cover both data streams.

---

## References

- [Install Kibana with Debian Package — Elastic Docs](https://www.elastic.co/guide/en/kibana/current/deb.html)
- [Kibana settings reference — Elastic Docs](https://www.elastic.co/guide/en/kibana/current/settings.html)
- [Configure security in Kibana — Elastic Docs](https://www.elastic.co/guide/en/kibana/current/using-kibana-with-security.html)
- [Kibana keystore — Elastic Docs](https://www.elastic.co/guide/en/kibana/current/secure-settings.html)
- [`elasticsearch-reset-password` — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/reset-password.html)
- [Built-in users — Elastic Docs](https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html)
- [Ubuntu Server 26.04 LTS Documentation](https://ubuntu.com/server/docs)
