# Building the Operational-Analytics Dashboard in Kibana — Step-by-Step UI Walkthrough

This is the click-by-click companion to **Part 12 → Section C5**. We build ten Lens panels on a single dashboard against the ECS-shaped indexes from Part 11 (`mule-logs-ecs` / `mule-traces-ecs`). The first panel is in full detail; the remaining nine are deltas — *"do exactly what you did in #1 except change these three things"* — because the muscle memory for Lens is what we are building.

Tested against **Kibana 8.x** (panel labels are stable enough that 7.17+ looks almost identical).

---

## Before you start — one-time setup

Five minutes that pay for themselves on every panel below.

### 0a. Confirm the data views exist with `@timestamp` as the time field

1. Top-left hamburger menu → **Stack Management**.
2. Under **Kibana** in the left rail → **Data views**.
3. You should see `mule-logs-ecs` and `mule-traces-ecs` listed. If a data view is missing:
   - Click **Create data view**.
   - **Name:** `Mule Logs` (the human-friendly name) · **Index pattern:** `mule-logs-ecs*` (trailing `*` future-proofs against rollover indexes like `mule-logs-ecs-2026.06.10`).
   - **Timestamp field:** pick `@timestamp` from the dropdown.
   - **Save data view to Kibana**.
   - Repeat for `Mule Traces` against `mule-traces-ecs*`.
4. For each existing data view, click into it and confirm the **Time field** at the top says `@timestamp`. If it says something else, click **Edit data view** → change it → save.

### 0b. Create the `duration_ms` runtime field on `Mule Traces`

You'll reuse this in #3, #4, #5, and #10. Do it once.

1. Stack Management → **Data views** → click **Mule Traces**.
2. Click **Add field** (top right).
3. **Name:** `duration_ms` · **Type:** `Double` · **Set value** toggle: ON.
4. In the script box:

```painless
if (doc['Duration'].size() != 0) {
  emit(doc['Duration'].value / 1000.0);
}
```

5. **Save**. The field appears under **Mule Traces**' field list with a runtime-field icon next to it.

### 0c. Create the `mule.thread.tier` runtime field on `Mule Logs` (for #10 only — skip if you're not doing #10)

Same path: Mule Logs data view → **Add field** → name `mule.thread.tier` · Type `Keyword` · Set value ON. Script:

```painless
if (doc['process.thread.name.keyword'].size() != 0) {
  def name = doc['process.thread.name.keyword'].value;
  def m = /\][^[]*\[(?:[^]]*)]\.[^.]+\.([A-Z_]+)/.matcher(name);
  if (m.find()) emit(m.group(1));
}
```

Save.

### 0d. Open or create the dashboard you'll add panels to

1. Top-left hamburger → **Analytics → Dashboard**.
2. **Create dashboard** (or open an existing one and click **Edit**).
3. Set the **time picker** at the top right to **Last 1 hour** while you build (you can change the default later in **Save**).

You should now be looking at a blank dashboard with **Add panel** in the top toolbar. From here, every viz below starts the same way: **Add panel → Visualization → Lens**.

---

## #1 — Error rate by application

This one is in full detail. Every other section below mirrors this flow.

1. **Add panel** (top of dashboard) → **Visualization** → **Lens**. The Lens editor opens.
2. **Top-left dropdown:** confirm it says **Mule Logs**. If it says Mule Traces, change it.
3. **Time range** at the top should already be **Last 1 hour** (inherited from the dashboard).
4. **Visualization type** (top-right of the editor): click the chart-type dropdown and pick **Line**.
5. **Drag fields onto the panel.** Lens has three drop zones in the right-hand column: **Horizontal axis**, **Vertical axis**, **Breakdown**.
   - Drag `@timestamp` from the field list (left rail) onto **Horizontal axis**. It should auto-set to interval `Auto`.
   - Now we set the Y axis as a Formula:
     - Drag **Records** (it's the first item under "Available fields" or "Special fields") onto **Vertical axis**.
     - Click the field that just landed on **Vertical axis** to open its config drawer.
     - At the top of the drawer, click the function dropdown (it currently says **Count**) → scroll to **Formula** at the bottom of the function list → click it.
     - In the Formula box that appears, paste:
       ```text
       count(kql='log.level : "ERROR"') / count() * 100
       ```
     - **Display name:** `Error rate (%)`.
     - **Format:** Percent (or leave Number — both display 0–100 sanely; Percent divides by 100, so if you use it, drop the `* 100` from the formula).
     - Close the drawer.
   - Drag `service.name` (find it in the left rail) onto **Breakdown**.
     - Click the field in **Breakdown** → **Number of values**: `10` → **Group remaining values as 'Other'**: ON.
     - Close the drawer.
6. **Title the panel.** Top of the Lens editor: click the title bar (**"Untitled"**) → type `Error rate (%) by service`.
7. **Save and return** (top right). You'll land back on the dashboard with the new panel.
8. **Resize:** drag the panel's bottom-right corner to make it wide enough to read the legend.

You should see one line per service, Y axis 0–100, X axis last hour. If a service has zero traffic in the window, its line just doesn't appear — that's fine.

> [!TIP]
> The first time you build a Formula, autocomplete suggests every available field. If `log.level` doesn't show up, the data view's field list is stale — exit Lens, go to Data views → Mule Logs → click **Refresh field list**, retry.

---

## #2 — Error trend over time per API layer

Same start as #1: **Add panel → Lens → Mule Logs**. Then:

1. **Visualization type:** **Area stacked**.
2. **KQL filter** (top of the Lens editor, the search bar): `service.environment : "prod" and log.level : "ERROR"`.
3. **Horizontal axis:** `@timestamp`, interval `Auto`.
4. **Vertical axis:** **Records** with function **Count**. Display name: `Errors`.
5. **Breakdown:** `api.layer`, **Number of values:** `3` (we only have three layers: experience/process/system).
6. Title: `Errors over time by api.layer`.
7. **Save and return**.

---

## #3 — Long-running transactions detection

Different data view this time.

1. **Add panel → Lens → Mule Traces**.
2. **Visualization type:** **Table**.
3. **KQL filter:** `Resource.deployment.environment : "prod" and Kind : "SPAN_KIND_SERVER" and Duration > 1500000`.
4. **Rows** (drag fields onto the **Rows** drop zone):
   - `TraceId.keyword`, top 50.
   - `Resource.service.name.keyword`, top 5.
   - `Attributes.http.route.keyword`, top 20.
5. **Metrics** (drag onto **Metrics** drop zone):
   - `duration_ms` with function **Maximum**, display name `Total ms`. (Use the runtime field from step 0b.)
   - `Attributes.client_id.keyword` with function **Last value**, display name `Client`.
6. Click the column header `Total ms` → choose **Sort: Descending**.
7. Title: `Slow transactions — top 50`.
8. **Save and return**.
9. **Wire the drilldown** (the click-through to logs):
   - On the dashboard, hover the panel → click the gear icon (top-right of the panel) → **Create drilldown**.
   - Pick **Go to URL**.
   - URL template:
     ```text
     /app/discover#/?_a=(index:'mule-logs-ecs',query:(language:kuery,query:'trace.id : "{{event.value}}"'),sort:!(!('@timestamp',asc)))&_g=(time:(from:now-24h,to:now))
     ```
   - **Trigger:** Single click on a value.
   - **Encode URL:** ON.
   - **Save**.
   - Configure on the **`TraceId` column**, not the panel root — Lens shows a small drilldown indicator on the column header once the drilldown is attached.

---

## #4 — Throughput per application (TPS trend)

1. **Add panel → Lens → Mule Traces**.
2. **Visualization type:** **Line**.
3. **KQL filter:** `Resource.deployment.environment : "prod" and Kind : "SPAN_KIND_SERVER"`.
4. **Horizontal axis:** `@timestamp`, interval `Auto` (Lens picks a sensible bucket; for true per-second TPS see the tip below).
5. **Vertical axis:** **Records** → function **Count**. Display name: `Requests`.
6. **Breakdown:** `Resource.service.name.keyword`, top 5.
7. Title: `Requests per service over time`.
8. **Save and return**.

> [!TIP]
> For true TPS (requests per second) rather than per-bucket count, change the Vertical axis to a Formula: `count() / 60` if your bucket interval is 1 minute, or wrap in `normalize_by_unit(count(), unit='s')` on Lens versions that support it. The auto interval is fine for most dashboards — humans read trend, not absolute rate.

---

## #5 — Throughput per replica (load distribution)

1. **Add panel → Lens → Mule Traces**.
2. **Visualization type:** **Line**.
3. **KQL filter:** `Resource.deployment.environment : "prod" and Kind : "SPAN_KIND_SERVER" and Resource.service.name : "process-payments-sepa"`.
4. **Horizontal axis:** `@timestamp`.
5. **Vertical axis:** **Records** → function **Count**. Display name: `Requests`.
6. **Breakdown:** `Resource.workerId.keyword`, top 10.
7. Title: `process-payments-sepa — load by replica`.
8. **Save and return**.

Then a companion panel for "how many replicas are alive":

1. **Add panel → Lens → Mule Traces**.
2. **Visualization type:** **Metric**.
3. **KQL filter:** same as above.
4. **Primary metric:** `Resource.workerId.keyword` → function **Unique count** (cardinality). Display name: `Active replicas`.
5. Title: `Active replicas (now)`.
6. **Save and return**.

---

## #6 — Errors by application version

1. **Add panel → Lens → Mule Logs**.
2. **Visualization type:** **Bar vertical**.
3. **KQL filter:** `service.environment : "prod" and log.level : "ERROR"`.
4. **Horizontal axis:** `service.version.keyword`, top 10. (Note: this is the X axis, but it's categorical — Lens calls the slot "Horizontal axis" regardless.)
5. **Vertical axis:** **Records** → function **Count**. Display name: `Errors`.
6. **Breakdown:** `service.name.keyword`, top 5.
7. Title: `Errors by service.version`.
8. **Save and return**.

---

## #7 — Error rate by flow

1. **Add panel → Lens → Mule Logs**.
2. **Visualization type:** **Bar horizontal**.
3. **KQL filter** (top bar): `service.environment : "prod"`.
4. **Vertical axis** (which Lens uses for *rows* on a horizontal bar): `mule.flow.processor_path.keyword`, top 20.
5. **Horizontal axis** (which Lens uses for *metrics* on a horizontal bar): three metrics — drag **Records** three times into the metric slot, configure each:
   - First metric: function **Count**, KQL filter on the metric drawer: `log.level : "ERROR"`. Display name: `Errors`.
   - Second metric: function **Count**, no filter. Display name: `Total`.
   - Third metric: function **Formula** with `count(kql='log.level : "ERROR"') / count() * 100`. Display name: `Error rate (%)`.
6. Sort the chart by `Error rate (%)` descending: click the third metric in the side drawer → **Rank by:** `Error rate (%)`, **Direction:** Descending.
7. Title: `Error rate by flow processor`.
8. **Save and return**.

> [!TIP]
> A separate metric KQL inside a metric drawer is *additional to* the panel-level KQL filter. The panel-level filter (`service.environment : "prod"`) applies to all three metrics; each metric's individual KQL filters down further. That's how you get errors and total in one panel without two queries.

---

## #8 — Top consumers by `client_id`

1. **Add panel → Lens → Mule Logs**.
2. **Visualization type:** **Bar horizontal**.
3. **KQL filter:** `service.environment : "prod"`.
4. **Vertical axis** (rows): `client_id.keyword`, top 20. Optionally add `service.name.keyword` as a second-level row if a single client calls many services.
5. **Horizontal axis** (metrics):
   - First: `trace.id.keyword` → function **Unique count**. Display name: `Requests`.
   - Second: **Records** → function **Count**. Display name: `Log volume`.
6. Sort by `Requests` descending: click the first metric → **Rank by:** `Requests`, **Direction:** Descending.
7. Title: `Top consumers by client_id`.
8. **Save and return**.

---

## #9 — Consumer error rate

1. **Add panel → Lens → Mule Logs**.
2. **Visualization type:** **Table**.
3. **KQL filter:** `service.environment : "prod"`.
4. **Rows:** `client_id.keyword`, top 20.
5. **Metrics:**
   - First: function **Count** with metric-level KQL `log.level : "ERROR"`. Display name: `Errors`.
   - Second: function **Count**, no filter. Display name: `Total`.
   - Third: function **Formula** with `count(kql='log.level : "ERROR"') / count() * 100`. Display name: `Error rate (%)`.
6. Sort by `Error rate (%)` descending.
7. **Filter out tiny clients** (avoids 100% rates from a client with 1 request):
   - In the row drawer for `client_id.keyword`, expand **Advanced** → **Min documents per term**: `100` (or whatever your noise threshold is).
8. Title: `Client error rate`.
9. **Save and return**.

---

## #10 — Detect thread saturation patterns

Three panels — one per visualization. Each is its own **Add panel → Lens** flow.

### 10a. Concurrent active threads (line)

1. **Mule Logs**, type **Line**.
2. **KQL:** `service.environment : "prod"`.
3. **X axis:** `@timestamp`.
4. **Y axis:** `process.thread.id` → function **Unique count**. Display name: `Active threads`.
5. **Breakdown:** `service.name.keyword`, top 5.
6. Title: `Active threads over time`.

### 10b. Thread reuse rate (line)

1. **Mule Logs**, type **Line**.
2. **KQL:** `service.environment : "prod"`.
3. **X axis:** `@timestamp`.
4. **Y axis:** function **Formula** with `count() / unique_count(process.thread.id)`. Display name: `Logs per thread`.
5. **Breakdown:** `service.name.keyword`, top 5.
6. Title: `Thread reuse rate`.

### 10c. Long-running thread pinning (table)

1. **Mule Logs**, type **Table**.
2. **KQL:** `service.environment : "prod"` and time picker last 5 min.
3. **Rows:** `process.thread.name.keyword`, top 50.
4. **Metric:** **Records** → function **Count**.
5. Sort by Count descending.
6. Title: `Most active threads (last 5 min)`.

### 10d. Bonus — saturation by thread tier (bar vertical)

If you set up the `mule.thread.tier` runtime field in step 0c:

1. **Mule Logs**, type **Bar vertical**.
2. **KQL:** `service.environment : "prod"`.
3. **X axis:** `mule.thread.tier`, top 10.
4. **Y axis:** `process.thread.id` → function **Unique count**.
5. Title: `Threads in use by pool tier`.

---

## Final saves and one piece of advice

Once you've built all the panels:

1. On the dashboard, click **Save** at the top right.
2. **Title:** `Mule Operational Analytics`.
3. **Tags:** add `mule`, `observability` if you have those tags configured.
4. **Time-restore:** **ON** if you want the dashboard to remember its time range. Pick **OFF** if you want it to inherit whatever the user has set globally.
5. **Save**.

To export the whole dashboard for sharing or version-control: **Stack Management → Saved Objects** → tick the dashboard + its Lens panels + the data views + the runtime fields → **Export** → toggle **Include related objects: ON** → save the NDJSON. That's the artifact you'd commit to the repo as part of Part 8 / Part 13's flow.

> [!IMPORTANT]
> **Always export with "Include related objects" on.** A bare dashboard NDJSON references the data views and runtime fields by ID; importing into a fresh cluster without the related objects gives a dashboard that opens with every panel showing *"data view not found."* The "Include related objects" toggle is the difference between a portable export and an export that only works in your cluster.

---

## See also

- **Part 12 → Section A** — the underlying KQL recipes each panel above is built on.
- **Part 12 → Section B** — alert rules that mirror panels #1, #3, #4, #6, #9 (so a dashboard finding becomes an automated page).
- **Part 12 → Section C5** — the panel inventory in the post itself; treat this file as the click-by-click expansion of that table.
- **Part 8** — the original unified dashboard with the cross-index drilldown pattern reused in #3.
- **Part 13** — exporting the dashboard NDJSON as a CI-deployable artifact.
