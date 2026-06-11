---
title: "Automating Mule Observability Properties — CI Profiles, Secrets, and GitHub Actions — Part 13"
slug: "mule-ci-cd-otel-properties-maven-profiles-github-actions"
description: "Automate the eleven Part 9 observability properties end-to-end — Maven profiles per environment, Anypoint Secure Properties for credentials, and a GitHub Actions workflow that deploys dev → qa → prod with approval gates."
author: "Gonzalo Marcos"
date: 2026-06-10
status: not validated
lang: en
category: devops
tags:
  - mule-runtime
  - cloudhub-2
  - runtime-fabric
  - ci-cd
  - github-actions
  - maven
  - mule-maven-plugin
  - secrets-management
  - english
  - series
series: "Sending Logs and Distributed Traces to Elastic"
series_part: 13
type: tutorial
difficulty: advanced
read_time: 17
mule_version: "4.11"
platform:
  - anypoint-platform
  - cloudhub-2
  - runtime-fabric
canonical_url: ""
---

![Draft](https://img.shields.io/badge/Status-Draft-7f8c8d) ![English](https://img.shields.io/badge/Lang-English-4a4a4a) ![Mule 4.11](https://img.shields.io/badge/Mule_Runtime-4.11-00A0DF?logo=mulesoft&logoColor=white) ![CloudHub 2.0](https://img.shields.io/badge/Platform-CloudHub_2.0-00A0DF?logo=mulesoft&logoColor=white) ![Runtime Fabric](https://img.shields.io/badge/Platform-Runtime_Fabric-00A0DF?logo=mulesoft&logoColor=white) ![Anypoint Platform](https://img.shields.io/badge/Platform-Anypoint_Platform-00A0DF?logo=mulesoft&logoColor=white) ![Advanced](https://img.shields.io/badge/Level-Advanced-e74c3c) ![Tutorial](https://img.shields.io/badge/Type-Tutorial-8e44ad) ![Series](https://img.shields.io/badge/Series-Sending_Logs_and_Distributed_Traces_to_Elastic-16a085) ![Part 13](https://img.shields.io/badge/Part-13-16a085) ![17 min](https://img.shields.io/badge/Read_Time-17_min-lightgrey)

# Automating Mule Observability Properties — CI Profiles, Secrets, and GitHub Actions — Part 13

By the end of [Part 12](#) we had a fully shaped Elastic stack — KQL recipes, alerts, dashboards — operating against the schema we built in Parts 9–11. Every piece of it depended on a Mule app being deployed to CloudHub 2.0 or Runtime Fabric with **the right Runtime Manager properties set correctly**. So far we have done that by clicking through the Properties tab in the Anypoint UI: fine for one or two demos, fragile for anything resembling production. The values are typed by humans, scope-bound to one app, gone the moment someone clicks **Delete** by accident, and impossible to audit after the fact.

This post automates that step. We will refactor the Mule project's `pom.xml` to declare every observability property as a placeholder, add **Maven profiles** per environment so the values diverge by `dev` / `qa` / `prod`, store the `mule-logger` password in **Anypoint Secure Properties** (encrypted at rest, decrypted at app boot), and wire a **GitHub Actions** workflow that promotes a build through dev → qa → prod with manual approval gates between stages. By the end, deploying to prod is a click on **Approve** in the Actions tab — every property identical to what passed qa, every credential pulled from a vault, every value visible in the run log except the secret.

The pattern works equally on GitLab CI, Jenkins, Azure Pipelines, or anything else that has variables, secrets, and approvals. GitHub Actions is the example because the YAML is short and most readers can run it for free.

> 🔗 **Series:** [Sending Logs and Distributed Traces to Elastic](#) · **Previous:** [Part 12 — Operating the Stack: KQL Recipes, Alerts, and Dashboards](#) · **Next:** Series wrap-up

> [!WARNING]
> **HTTP-only, demo-grade.** Same posture as the rest of the series. The CI workflow below uses GitHub Actions secrets — fine for the demo. Production-grade alternatives (HashiCorp Vault, AWS Secrets Manager) plug in the same way; that integration is out of scope here.

---

## What We Will Cover

- A classification of the eleven observability properties by **who owns the value**: app/repo, pipeline/env, or secret.
- **`pom.xml` refactor** — every value as a `${...}` placeholder; constants in `<properties>`, deployment-time values in Maven profiles.
- **Maven profiles per environment** — one profile per `dev` / `qa` / `prod`, env-varying values version-controlled.
- **Anypoint Secure Properties** for the `mule-logger` password — encrypted blob in source, decrypted at app boot.
- A **GitHub Actions workflow** that builds once, promotes the same artifact through dev → qa → prod, gates each promotion on approval, and pulls secrets per environment.
- Verification — read each deployed app's Properties tab, confirm every value matches what the build produced, confirm no secret is on disk in cleartext.

---

## Prerequisites

- The Mule 4.11 project from [Parts 5–11](#), already deployed by hand to at least one environment.
- A GitHub repo for the project, with **GitHub Environments** configured for `dev`, `qa`, `prod` (Settings → Environments). Required-reviewer rules on `qa` and `prod` are recommended.
- Anypoint Platform credentials — preferably a **connected app** (server-to-server) rather than a user account, with the Manage Applications scope. Username/password also works.
- The OTel Collector deployed in each environment from [Part 5](#), with reachable hostnames or DNS names per environment.
- Local **Maven 3.9+** and **JDK 17** for testing the profiles before pushing.

---

## Property Ownership — the Classification That Drives Everything

Before any YAML or POM edit, classify the eleven Part 9 properties by **who owns the value**. The classification is the entire design — once it is clear, the mechanism that follows is mechanical.

| Property | Owner | Where it should live |
| --- | --- | --- |
| `mule.openTelemetry.exporter.resource.service.name` | App / repo | `pom.xml` `${project.artifactId}` (does not vary across environments) |
| `mule.openTelemetry.exporter.resource.service.namespace` | App / repo | `pom.xml` constant |
| `service.version` (in `resource.attributes`) | Build | `${project.version}` — already in the build |
| `api.layer` | App / repo | `pom.xml` constant per app (`process` for the SEPA service) |
| `business.domain` | App / repo | `pom.xml` constant per app (`payments`) |
| `mule.runtime.version` | Pipeline | Per-env Maven profile (kept in lockstep with the deployment's `runtimeVersion`) |
| `deployment.environment` | Pipeline | Per-env Maven profile (`dev` / `qa` / `prod`) |
| `deployment.target` | Pipeline | Per-env Maven profile (constant `runtime-fabric` for our example) |
| `deployment.region` | Pipeline | Per-env Maven profile |
| OTel collector endpoints (`logging.exporter.endpoint`, `tracer.exporter.endpoint`) | Pipeline / per-env | Per-env Maven profile (different DNS per environment) |
| Constants (`mule.put.trace.id.and.span.id.in.mdc=true`, `tracer.exporter.sampler=alwaysOn`, `logging.exporter.level=INFO`) | Constant | `pom.xml` `<properties>` block |
| `mule-logger` password (only if Path B / a future Filebeat path needs it) | Secret | **Anypoint Secure Properties** (or a vault, retrieved at deploy time) |
| Anypoint username/password (for the deploy itself) | Secret | **GitHub Actions secret**, scoped per environment |

Three buckets emerge:

- **App-owned, repo-versioned.** `service.name`, `service.namespace`, `api.layer`, `business.domain`, plus the constants. Belong in `pom.xml`. Travel with the artifact. Changing them is a code review.
- **Pipeline-owned, env-varying.** Everything that differs per environment. Belong in Maven profiles checked into the repo (visible diffs, version-controlled, no secrets). For values that change too often to commit, the pipeline overrides them with `-D` flags.
- **Secrets.** Anything that is a credential. Belong in a secret store — never in the POM, never in `mule-app.properties`, never in `pom.xml` `<properties>`. Referenced by name from the pipeline or via Anypoint Secure Properties on the Mule side.

---

## Overview

```
┌────────────────────────────────────────────────────────────────────┐
│  Source of truth                                                   │
│                                                                    │
│  pom.xml                                                           │
│   ├─ <properties>     — app-owned constants (service.name, ...)    │
│   ├─ <profiles>                                                    │
│   │   ├─ dev          — env-varying values, dev                    │
│   │   ├─ qa           — env-varying values, qa                     │
│   │   └─ prod         — env-varying values, prod                   │
│   └─ <plugin> mule-maven-plugin                                    │
│       └─ <cloudHub2Deployment>                                     │
│           └─ <properties>   — every value is ${placeholder}        │
└────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌────────────────────────────────────────────────────────────────────┐
│  GitHub Actions workflow — .github/workflows/deploy.yml            │
│                                                                    │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐               │
│  │  build      │──▶│  deploy-dev │──▶│  deploy-qa  │──▶ deploy-prod│
│  │  (artifact) │   │ -Pdev       │   │ -Pqa        │   -Pprod      │
│  │             │   │             │   │ approval    │   approval    │
│  └─────────────┘   └─────────────┘   └─────────────┘               │
│                                                                    │
│  Secrets per Environment (Settings → Environments):                │
│   ANYPOINT_USERNAME, ANYPOINT_PASSWORD, MULE_LOGGER_PASSWORD,      │
│   SECURE_PROPERTIES_KEY                                            │
└────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
                        Anypoint Runtime Manager
                                  │
                                  ▼
                       Mule app on CH2 / RTF
                  with all 11 properties set correctly
```

---

## Table of Contents

1. [Step 1 — Refactor `pom.xml`: Constants and Placeholders](#step-1--refactor-pomxml-constants-and-placeholders)
2. [Step 2 — Add One Maven Profile per Environment](#step-2--add-one-maven-profile-per-environment)
3. [Step 3 — Refactor the Deployment Plugin to Use Placeholders](#step-3--refactor-the-deployment-plugin-to-use-placeholders)
4. [Step 4 — Encrypt the `mule-logger` Password with Anypoint Secure Properties](#step-4--encrypt-the-mule-logger-password-with-anypoint-secure-properties)
5. [Step 5 — Configure GitHub Environments and Secrets](#step-5--configure-github-environments-and-secrets)
6. [Step 6 — Write the GitHub Actions Workflow](#step-6--write-the-github-actions-workflow)
7. [Step 7 — Trigger the Pipeline End-to-End](#step-7--trigger-the-pipeline-end-to-end)
8. [Verification](#verification)
9. [Troubleshooting](#troubleshooting)

---

### Step 1 — Refactor `pom.xml`: Constants and Placeholders

We will move every observability constant into the POM's top-level `<properties>` block. These are the values that **do not change** across environments — the artifact identity and the constants the spec dictates.

📄 `pom.xml` — top-level `<properties>` section

```xml
<properties>
    <app.runtime>4.12.0</app.runtime>
    <mule.maven.plugin.version>4.4.0</mule.maven.plugin.version>

    <!-- App-owned observability constants — same value in every environment -->
    <obs.service.namespace>retail-banking</obs.service.namespace>
    <obs.api.layer>process</obs.api.layer>
    <obs.business.domain>payments</obs.business.domain>

    <!-- OTel exporter constants — same in every environment -->
    <obs.tracer.sampler>alwaysOn</obs.tracer.sampler>
    <obs.logging.level>INFO</obs.logging.level>
    <obs.put.trace.in.mdc>true</obs.put.trace.in.mdc>
</properties>
```

A few things worth noticing:

- **`obs.*` prefix.** Every observability-related Maven property gets a single namespace so the POM stays readable. Without it, `service.namespace` and `business.domain` would be ambiguous against any Maven-built-in property.
- **`obs.service.name` is intentionally absent.** That value is `${project.artifactId}` already — referencing it as a separate property would mean two sources of truth for the same value. We will inline `${project.artifactId}` directly in Step 3.
- **Constants only.** No environment-varying values here. Anything that changes between `dev` and `prod` belongs in a profile (Step 2).

---

### Step 2 — Add One Maven Profile per Environment

The values that vary between `dev` / `qa` / `prod` go into Maven profiles in the same `pom.xml`. Each profile sets its own values for the placeholders the deployment plugin will reference in Step 3.

📄 `pom.xml` — `<profiles>` section

```xml
<profiles>
    <profile>
        <id>dev</id>
        <properties>
            <env.name>dev</env.name>
            <obs.deployment.environment>dev</obs.deployment.environment>
            <obs.deployment.target>runtime-fabric</obs.deployment.target>
            <obs.deployment.region>eu-central-1</obs.deployment.region>
            <obs.runtime.version>${app.runtime}</obs.runtime.version>
            <obs.collector.logs.endpoint>http://otel-dev.internal:4318/v1/logs</obs.collector.logs.endpoint>
            <obs.collector.traces.endpoint>http://otel-dev.internal:4318/v1/traces</obs.collector.traces.endpoint>

            <ch2.target>${anypoint.bg.dev.private-space.id}</ch2.target>
            <anypoint.environment>Sandbox</anypoint.environment>
        </properties>
    </profile>

    <profile>
        <id>qa</id>
        <properties>
            <env.name>qa</env.name>
            <obs.deployment.environment>qa</obs.deployment.environment>
            <obs.deployment.target>runtime-fabric</obs.deployment.target>
            <obs.deployment.region>eu-central-1</obs.deployment.region>
            <obs.runtime.version>${app.runtime}</obs.runtime.version>
            <obs.collector.logs.endpoint>http://otel-qa.internal:4318/v1/logs</obs.collector.logs.endpoint>
            <obs.collector.traces.endpoint>http://otel-qa.internal:4318/v1/traces</obs.collector.traces.endpoint>

            <ch2.target>${anypoint.bg.qa.private-space.id}</ch2.target>
            <anypoint.environment>QA</anypoint.environment>
        </properties>
    </profile>

    <profile>
        <id>prod</id>
        <properties>
            <env.name>prod</env.name>
            <obs.deployment.environment>prod</obs.deployment.environment>
            <obs.deployment.target>runtime-fabric</obs.deployment.target>
            <obs.deployment.region>eu-central-1</obs.deployment.region>
            <obs.runtime.version>${app.runtime}</obs.runtime.version>
            <obs.collector.logs.endpoint>http://otel-prod.internal:4318/v1/logs</obs.collector.logs.endpoint>
            <obs.collector.traces.endpoint>http://otel-prod.internal:4318/v1/traces</obs.collector.traces.endpoint>

            <ch2.target>${anypoint.bg.prod.private-space.id}</ch2.target>
            <anypoint.environment>Production</anypoint.environment>
        </properties>
    </profile>
</profiles>
```

> [!IMPORTANT]
> **Diff-friendly is the point.** A side-by-side diff of `dev` and `prod` should make every difference between the two environments visible in five seconds. If the same value is repeated three times across the three profiles, that means it is actually a constant — pull it up into the top-level `<properties>` block. The principle is: profiles only carry the values that *differ*.

> [!TIP]
> Some teams prefer **one Maven profile per environment + one external `properties` file per environment** (Pattern 3 from the chat thread that triggered this post). That is the right choice once the per-env value count crosses ~15 or once a non-developer team needs to edit them without touching `pom.xml`. For our eleven Mule observability properties at three environments, profiles in the POM stay readable.

---

### Step 3 — Refactor the Deployment Plugin to Use Placeholders

Now we wire the Mule Maven Plugin. **Every** value in `<cloudHub2Deployment>` becomes a `${placeholder}` — none are literal.

📄 `pom.xml` — `mule-maven-plugin` configuration

```xml
<plugin>
    <groupId>org.mule.tools.maven</groupId>
    <artifactId>mule-maven-plugin</artifactId>
    <version>${mule.maven.plugin.version}</version>
    <extensions>true</extensions>
    <configuration>
        <cloudHub2Deployment>
            <runtimeVersion>${obs.runtime.version}</runtimeVersion>
            <muleVersion>${obs.runtime.version}</muleVersion>

            <username>${anypoint.username}</username>
            <password>${anypoint.password}</password>
            <environment>${anypoint.environment}</environment>
            <businessGroupId>${anypoint.bg.id}</businessGroupId>
            <target>${ch2.target}</target>

            <applicationName>${project.artifactId}-${env.name}</applicationName>
            <replicas>1</replicas>
            <vCores>0.1</vCores>

            <properties>
                <!-- Identity (Resource.service.*) — app-owned, no env override -->
                <mule.openTelemetry.exporter.resource.service.name>${project.artifactId}</mule.openTelemetry.exporter.resource.service.name>
                <mule.openTelemetry.exporter.resource.service.namespace>${obs.service.namespace}</mule.openTelemetry.exporter.resource.service.namespace>

                <!-- The big resource.attributes string — built from the per-env profile -->
                <mule.openTelemetry.exporter.resource.attributes>service.version=${project.version},deployment.environment=${obs.deployment.environment},deployment.target=${obs.deployment.target},deployment.region=${obs.deployment.region},api.layer=${obs.api.layer},business.domain=${obs.business.domain},mule.runtime.version=${obs.runtime.version}</mule.openTelemetry.exporter.resource.attributes>

                <!-- OTel logs exporter -->
                <mule.openTelemetry.logging.exporter.enabled>true</mule.openTelemetry.logging.exporter.enabled>
                <mule.openTelemetry.logging.exporter.type>HTTP</mule.openTelemetry.logging.exporter.type>
                <mule.openTelemetry.logging.exporter.endpoint>${obs.collector.logs.endpoint}</mule.openTelemetry.logging.exporter.endpoint>
                <mule.openTelemetry.logging.exporter.level>${obs.logging.level}</mule.openTelemetry.logging.exporter.level>

                <!-- OTel tracer exporter -->
                <mule.openTelemetry.tracer.exporter.enabled>true</mule.openTelemetry.tracer.exporter.enabled>
                <mule.openTelemetry.tracer.exporter.type>HTTP</mule.openTelemetry.tracer.exporter.type>
                <mule.openTelemetry.tracer.exporter.endpoint>${obs.collector.traces.endpoint}</mule.openTelemetry.tracer.exporter.endpoint>
                <mule.openTelemetry.tracer.exporter.sampler>${obs.tracer.sampler}</mule.openTelemetry.tracer.exporter.sampler>

                <!-- MDC trace bridge -->
                <mule.put.trace.id.and.span.id.in.mdc>${obs.put.trace.in.mdc}</mule.put.trace.id.and.span.id.in.mdc>
            </properties>
        </cloudHub2Deployment>
    </configuration>
</plugin>
```

📄 Full file: [./assets/process-payments-sepa/pom.xml](./assets/process-payments-sepa/pom.xml)

> [!IMPORTANT]
> **The `<applicationName>${project.artifactId}-${env.name}</applicationName>` line is intentional.** It produces `process-payments-sepa-dev`, `-qa`, `-prod` as distinct Anypoint applications — same artifact, different deployment targets. Without the `${env.name}` suffix, deploying `qa` would overwrite `dev` because they would share the same application name. This is the most common cause of *"my dev environment disappeared"* in CI-driven Mule deployments.

> [!TIP]
> Sanity-test the placeholder resolution locally before any CI work. From the project root:
>
> ```bash
> mvn help:effective-pom -Pprod | grep -A 30 cloudHub2Deployment
> ```
>
> Every `<...>...</...>` line in the printed effective POM should contain a literal value, not a `${placeholder}`. Any unresolved placeholder there is a value the pipeline has not been told about.

---

### Step 4 — Encrypt the `mule-logger` Password with Anypoint Secure Properties

If the app talks to Elasticsearch directly (Part 4's Log4j2 path) or to any other service that needs a password, the credential should ride in the deployable artifact as an **encrypted blob** — not a Runtime Manager property in cleartext.

> [!NOTE]
> **The Direct Telemetry Stream path (Parts 5–11) does not need `mule-logger` on the Mule side.** Mule emits OTLP to the OTel Collector, and the collector authenticates to Elasticsearch from its own `MULE_LOGGER_PASSWORD` env var (Part 5 Step 3). If your Mule apps only ship telemetry through the collector, skip this step. Keep reading if you have a Part 4-style HTTP appender, a database connector, or any other Mule-side credential.

#### 4.1 — Add the Secure Configuration module

In Studio: **Mule Palette → Search in Exchange → Mule Secure Configuration Properties Extension → Add to project**. The Maven plugin lands automatically in `pom.xml`.

#### 4.2 — Create an encrypted properties file

Create `src/main/resources/properties/secure-${env.name}.properties` for each environment:

📄 `src/main/resources/properties/secure-prod.properties`

```properties
elastic.password=![<encrypted-blob-here>]
```

The `![...]` syntax is what tells the Secure Configuration Properties module that the value is encrypted. Generate the encrypted value with the **Mule Secure Properties Tool** (downloadable from Anypoint, or via `mvn` plugin):

```bash
java -cp secure-properties-tool.jar com.mulesoft.tools.SecurePropertiesTool \
    string \
    encrypt \
    AES \
    CBC \
    "${SECURE_PROPERTIES_KEY}" \
    "${ELASTIC_PASSWORD_PLAINTEXT}"
```

The tool prints a base64-encoded blob — that is what goes inside `![...]` in the file.

#### 4.3 — Reference the encrypted file from the Mule app config

📄 `src/main/mule/global.xml`

```xml
<secure-properties:config name="Secure_Properties"
                          file="properties/secure-${env.name}.properties"
                          key="${secure.properties.key}">
    <secure-properties:encrypt algorithm="AES" mode="CBC" />
</secure-properties:config>
```

The `${env.name}` pattern resolves at deployment time — `dev` reads `secure-dev.properties`, `prod` reads `secure-prod.properties`. The `${secure.properties.key}` placeholder is the AES key Mule uses to decrypt the file at boot — set as a Runtime Manager **Secure** property at deploy time, never in source.

#### 4.4 — Use the decrypted value

Anywhere in the Mule app, reference the property as `${secure::elastic.password}`:

```xml
<http:request-config name="Elastic_Request_config">
    <http:request-connection host="elastic.internal"
                             port="9200"
                             username="mule-logger"
                             password="${secure::elastic.password}"/>
</http:request-config>
```

The `secure::` prefix is what tells Mule to fetch the value from the Secure Properties config and decrypt it before using it.

> [!IMPORTANT]
> **The AES key is the only thing the pipeline must keep secret.** Once an attacker has the encryption key, every encrypted blob in the repo is decryptable in seconds. Store the key in **Anypoint Runtime Manager as a Secure property** (separate from the regular Properties tab). The CI passes it via `mvn deploy` arguments, never writes it to disk, and reads it from a vault-grade store.

---

### Step 5 — Configure GitHub Environments and Secrets

GitHub Actions has two storage tiers for the values our workflow needs:

- **Variables** — visible in the run log, fine for non-sensitive values like Anypoint environment IDs.
- **Secrets** — masked in the run log, used for the credentials.

Both can be scoped to a **GitHub Environment** (`Settings → Environments`), which adds approval gates and per-environment isolation. We will use that for `qa` and `prod`.

#### 5.1 — Create the three Environments

In the repo: **Settings → Environments → New environment** for each of `dev`, `qa`, `prod`. Required-reviewers settings:

- `dev` — no approval (auto-deploy on every push to `main`).
- `qa` — at least one reviewer (a peer).
- `prod` — at least two reviewers (release manager + app owner).

#### 5.2 — Add per-environment secrets and variables

Inside each Environment's **Secrets** tab:

| Name | Type | Example value | Where it goes |
| --- | --- | --- | --- |
| `ANYPOINT_USERNAME` | Secret | `obsdeploy_dev` (connected-app client ID) | `mvn deploy -Danypoint.username=...` |
| `ANYPOINT_PASSWORD` | Secret | `<connected-app-client-secret>` | `mvn deploy -Danypoint.password=...` |
| `ANYPOINT_BG_ID` | Variable | `98b7f48b-...` | `mvn deploy -Danypoint.bg.id=...` |
| `CH2_PRIVATE_SPACE_ID` | Variable | `09584395-...` | resolves `${anypoint.bg.dev.private-space.id}` etc. |
| `SECURE_PROPERTIES_KEY` | Secret | the AES key from Step 4 | `mvn deploy -Dsecure.properties.key=...` |

Every value is per-environment — `dev` cannot accidentally use `prod` credentials because the `prod` secrets are simply not visible to a job running with `environment: dev`.

> [!IMPORTANT]
> **Use a connected app, not a user account.** Anypoint's connected-app credentials are scoped (Manage Applications, Read Organization), revocable per-environment, and survive a person leaving the team. A user account credential creates a dependency on the person who owns it. The Mule docs walk through creating one under **Access Management → Connected Apps**.

---

### Step 6 — Write the GitHub Actions Workflow

The workflow has four jobs: `build` (once, produces an artifact), `deploy-dev`, `deploy-qa`, `deploy-prod`. Each deploy job depends on the previous one and runs against its own GitHub Environment.

📄 `.github/workflows/deploy.yml`

```yaml
name: Build and deploy process-payments-sepa

on:
  push:
    branches: [main]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.read-version.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven

      - id: read-version
        name: Read project version from pom.xml
        run: |
          version=$(mvn help:evaluate -Dexpression=project.version -q -DforceStdout)
          echo "version=$version" >> $GITHUB_OUTPUT

      - name: Build the artifact
        run: mvn -B clean package -DskipTests

      - name: Upload the artifact
        uses: actions/upload-artifact@v4
        with:
          name: mule-app
          path: target/*.jar
          retention-days: 14

  deploy-dev:
    needs: build
    runs-on: ubuntu-latest
    environment: dev
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven

      - uses: actions/download-artifact@v4
        with:
          name: mule-app
          path: target

      - name: Deploy to dev
        run: |
          mvn -B -Pdev deploy -DmuleDeploy \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -Danypoint.bg.id=${{ vars.ANYPOINT_BG_ID }} \
            -Danypoint.bg.dev.private-space.id=${{ vars.CH2_PRIVATE_SPACE_ID }} \
            -Dsecure.properties.key=${{ secrets.SECURE_PROPERTIES_KEY }}

  deploy-qa:
    needs: deploy-dev
    runs-on: ubuntu-latest
    environment: qa     # GH Environments — gates on approval
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven

      - uses: actions/download-artifact@v4
        with:
          name: mule-app
          path: target

      - name: Deploy to qa
        run: |
          mvn -B -Pqa deploy -DmuleDeploy \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -Danypoint.bg.id=${{ vars.ANYPOINT_BG_ID }} \
            -Danypoint.bg.qa.private-space.id=${{ vars.CH2_PRIVATE_SPACE_ID }} \
            -Dsecure.properties.key=${{ secrets.SECURE_PROPERTIES_KEY }}

  deploy-prod:
    needs: deploy-qa
    runs-on: ubuntu-latest
    environment: prod   # GH Environments — gates on approval (typically two reviewers)
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'
          cache: maven

      - uses: actions/download-artifact@v4
        with:
          name: mule-app
          path: target

      - name: Deploy to prod
        run: |
          mvn -B -Pprod deploy -DmuleDeploy \
            -Danypoint.username=${{ secrets.ANYPOINT_USERNAME }} \
            -Danypoint.password=${{ secrets.ANYPOINT_PASSWORD }} \
            -Danypoint.bg.id=${{ vars.ANYPOINT_BG_ID }} \
            -Danypoint.bg.prod.private-space.id=${{ vars.CH2_PRIVATE_SPACE_ID }} \
            -Dsecure.properties.key=${{ secrets.SECURE_PROPERTIES_KEY }}
```

📄 Full file: [./assets/process-payments-sepa/.github/workflows/deploy.yml](./assets/process-payments-sepa/.github/workflows/deploy.yml)

A few things worth knowing about this workflow:

- **Build once, deploy three times.** The `build` job runs `mvn package` and uploads the artifact; each deploy job downloads the same `.jar`. The artifact is *identical* across environments — only the Maven profile and the credentials differ.
- **`environment: dev` / `qa` / `prod`** is what scopes secrets and triggers approval gates. A typo on this line is the difference between a clean release and an audit incident.
- **`needs:` chains the jobs.** `deploy-qa` waits for `deploy-dev`, `deploy-prod` waits for `deploy-qa`. Failure at any stage halts the pipeline.
- **Secrets are referenced as `${{ secrets.X }}`, variables as `${{ vars.X }}`.** GitHub masks secret values in the run log; variables are visible. Use `vars` for anything you want auditable in the run log (e.g. the BG ID), `secrets` for anything you don't (e.g. passwords and AES keys).
- **`-Pprod` activates the matching profile.** Without that flag, Maven would not know which profile's values to inject, and every placeholder in the deployment plugin would resolve to the empty string — which deploys the app, but with no observability properties. Silent failure mode worth catching with the `mvn help:effective-pom` check from Step 3.

> [!WARNING]
> **Do not use `secret-key.json` or any other on-disk credential format checked into the repo.** Even a `.gitignore`-protected file leaks the moment one person clones the repo and pushes a change to the gitignore by accident. GitHub Actions secrets stay out of `git` entirely; the runner injects them into env vars at job start and discards them at job end.

---

### Step 7 — Trigger the Pipeline End-to-End

A single push to `main` is now enough to:

1. Build the artifact once.
2. Deploy to `dev` automatically.
3. Wait at the `qa` gate for a reviewer's approval.
4. After approval, deploy to `qa`.
5. Wait at the `prod` gate for two reviewers' approvals.
6. After approval, deploy to `prod`.

```bash
git add pom.xml .github/workflows/deploy.yml src/main/resources/properties/secure-prod.properties
git commit -m "Wire CI for observability properties + secure props"
git push origin main
```

Watch the run in **Actions → Build and deploy process-payments-sepa**. The deploy logs print the resolved Maven property values (Mule Maven Plugin output) — every observability property should show its environment-specific value, every credential should show as `***` (masked).

> [!NOTE]
> *Screenshot — GitHub Actions run page showing all four jobs in the pipeline, with `deploy-dev` complete (green), `deploy-qa` waiting on approval (yellow), and `deploy-prod` queued.*

> [!TIP]
> The first run almost always fails on a typo somewhere — a missing `${...}` placeholder, a wrong Anypoint env ID, a missed CI variable. Log into Anypoint Runtime Manager → the deployed app → **Settings → Properties** and read the resolved values directly. That is the fastest way to find which placeholder did not resolve.

---

## Verification

Three checks to confirm the pipeline is wired end-to-end and the values land where they should.

**1. Every property resolved correctly in the deployed app.**

In **Anypoint Runtime Manager → Applications → process-payments-sepa-prod → Settings → Properties**, every observability property should show a literal value:

- `mule.openTelemetry.exporter.resource.service.name = process-payments-sepa`
- `mule.openTelemetry.exporter.resource.service.namespace = retail-banking`
- `mule.openTelemetry.exporter.resource.attributes = service.version=1.2.0,deployment.environment=prod,...`
- `mule.openTelemetry.logging.exporter.endpoint = http://otel-prod.internal:4318/v1/logs`
- ...and so on for every property in Step 3.

A property showing `${...}` is a placeholder that was never resolved — pipeline did not pass that flag.

**2. The GitHub Actions run log shows every secret as masked.**

Open the latest deploy run, expand the `Deploy to prod` step. Search the log for the literal value of any secret (`ANYPOINT_PASSWORD`, `SECURE_PROPERTIES_KEY`). It should appear as `***` everywhere. If the literal value appears anywhere, the workflow has a `echo` or `print` step that needs to be removed — credentials in CI logs are an audit incident.

**3. Logs and traces still arrive in Elastic with the right Resource fields.**

Hit the deployed app in `prod`, then in Kibana → Discover on `mule-logs-ecs`:

```
service.environment : "prod" and service.name : "process-payments-sepa"
```

The most recent document should carry `Resource.deployment.environment = "prod"`, `Resource.api.layer = "process"`, `Resource.business.domain = "payments"`, etc. — exactly what the `prod` Maven profile defined.

---

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Property tab shows literal `${obs.deployment.environment}` instead of `prod` | Maven profile not activated — the `-Pprod` flag is missing in the CI command | Add `-Pprod` (or `-Pdev`/`-Pqa`) to the `mvn deploy` step |
| Property tab shows the value but the app's logs do not carry it | Property name typo (camel-case `T` in `openTelemetry`) | Re-check the property names in the `<cloudHub2Deployment><properties>` block — Mule does not error on unknown property names, it just ignores them |
| `mvn help:effective-pom -Pprod` shows literal `${...}` for `${anypoint.bg.prod.private-space.id}` | The CI variable is not defined in the GitHub Environment for `prod` | **Settings → Environments → prod → Variables** — add the missing variable |
| GitHub Actions run fails with `Cannot resolve symbol Secrets` | Workflow has `${{ Secrets.X }}` instead of `${{ secrets.X }}` (case-sensitive) | Fix to lowercase `secrets` |
| `dev` deploy succeeds but `qa` deploy overwrites it | Application name is the same in both environments | Confirm `<applicationName>${project.artifactId}-${env.name}</applicationName>` and the `env.name` profile property |
| `mule-logger` password decryption fails at app boot — `Could not decrypt property elastic.password` | The `secure.properties.key` Runtime Manager property does not match the AES key used to encrypt the file | Re-encrypt the value with the actual AES key the runtime is configured with; pass `-Dsecure.properties.key=...` in CI |
| Deploy fails with `401 Unauthorized` to Anypoint | Connected-app credentials revoked or scoped wrong | Re-issue the connected-app secret in Anypoint Access Management; update the GitHub Environment secret |
| Deploys land on the wrong CH2 private space | `vars.CH2_PRIVATE_SPACE_ID` value mixed up between environments | The variable is per-Environment; verify each environment's value points at the matching private space |

---

## What We Covered

- Classified the eleven Part 9 properties by **who owns the value** — app/repo, pipeline/env, or secret — and let that classification dictate where each property lives.
- Refactored `pom.xml` to declare every observability value as a placeholder: constants in the top-level `<properties>` block, env-varying values in **Maven profiles** per `dev` / `qa` / `prod`.
- Wired the **Mule Maven Plugin's** `<cloudHub2Deployment><properties>` block to resolve every property from those placeholders, so a single `mvn -Pprod deploy` produces a fully-shaped Runtime Manager configuration.
- Encrypted the `mule-logger` password with **Anypoint Secure Properties** and made the AES key the only secret the pipeline needs to keep — every blob in the repo is useless without it.
- Wrote a **GitHub Actions workflow** that builds the artifact once and promotes it through dev → qa → prod with manual approval gates between stages, scoping secrets per environment via **GitHub Environments**.
- Verified the pipeline by checking property resolution in Runtime Manager, secret masking in the Actions log, and live data in Kibana.

We can now ship a Mule app that emits every observability field correctly, in every environment, with no manual property entry, with credentials never on disk in cleartext, with full audit history of who deployed what when. That is the promotion path.

> ➡️ **Coming next — series wrap-up.** With CI in place, we have closed the loop: cluster up, apps emitting, schema standardized, dashboards populated, alerts firing, deployment automated. The natural follow-up series — *"From OTel Collector to Elastic APM"* — swaps the collector for Elastic APM Server and unlocks Kibana's full Observability suite. Same Mule properties, same GitHub Actions workflow, one fewer component.

---

## References

- [Mule Maven Plugin — Deployment Configuration](https://docs.mulesoft.com/mule-runtime/latest/mmp-concept)
- [Mule Maven Plugin — CloudHub 2.0 Deployment](https://docs.mulesoft.com/mule-runtime/latest/deploy-to-cloudhub-2)
- [Anypoint Secure Configuration Properties Module](https://docs.mulesoft.com/mule-runtime/latest/secure-configuration-properties)
- [Connected Apps for Anypoint Platform](https://docs.mulesoft.com/access-management/connected-apps-overview)
- [GitHub Actions — Environments](https://docs.github.com/en/actions/managing-workflow-runs-and-deployments/managing-deployments/managing-environments-for-deployment)
- [GitHub Actions — Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Maven Profiles — Apache Maven Docs](https://maven.apache.org/guides/introduction/introduction-to-profiles.html)
