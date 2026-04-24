**Note**: this is a work in progress, please help fact check and let me know how to improve.

# OpenTelemetry APM UI — What Data Powers What

New Relic does not display raw OpenTelemetry data directly in the APM UI. Instead, it **synthesizes a set of `apm.service.*` metrics** from OTel data, then uses those to drive the UI — the same way the New Relic language agents do. The original OTel data is still available for custom queries.

---

## How it works (the short version)

```
 OTel instrumentation
          │
          ▼
  OpenTelemetry Metrics & Spans   ────────────────────────────────┐
    (http.server.request.duration,                                │
     db.client.operation.duration, spans, etc.)                   │
          │                                                       │
          ▼                                                       │
  New Relic synthesizes                                           │
    apm.service.transaction.duration                              │
    apm.service.transaction.overview                              │
    apm.service.external.host.duration                            │
    apm.service.datastore.operation.duration                      │
          │                                                       │
          ▼                                                       ▼
    APM UI pages                                       Original data still
    (Summary, Transactions,                            available in custom
     Databases, External Services)                     dashboards & NRQL

```

**Metrics are preferred over spans** because spans are typically sampled and may not represent 100% of traffic.

---

## Quick reference: What does each UI section need?

| UI Section                                | Powered by                                 | Required OTel source                              |
| ----------------------------------------- | ------------------------------------------ | ------------------------------------------------- |
| **Summary**                               | `apm.service.transaction.duration`         | HTTP or RPC server metrics (see below)            |
| **Transactions**                          | `apm.service.transaction.duration`         | HTTP or RPC server metrics                        |
| **Transaction segment breakdown**         | `apm.service.transaction.overview`         | Spans only — server + nested client spans         |
| **Databases**                             | `apm.service.datastore.operation.duration` | DB metrics (v1.33+) or DB client spans (fallback) |
| **DB "Time consumption by caller"**       | `apm.service.transaction.overview`         | Spans only                                        |
| **External Services**                     | `apm.service.external.host.duration`       | HTTP/RPC client metrics                           |
| **External "Time consumption by caller"** | `apm.service.transaction.overview`         | Spans only                                        |
| **Distributed Tracing**                   | —                                          | Spans                                             |
| **Errors Inbox**                          | —                                          | Spans with `otel.status_code = ERROR`             |
| **Logs**                                  | —                                          | Logs                                              |
| **JVM Runtime**                           | —                                          | JVM runtime metrics (OTel Java agent)             |
| **Go Runtime**                            | —                                          | Go runtime metrics (OTel Go contrib)              |

---

## Metric-by-metric breakdown

### `apm.service.transaction.duration`

**Powers:** Summary page, Transactions page, error rate

Synthesized from your **server-side HTTP or RPC metrics**.

| Source OTel metric                                                                                                              | Required              | Source attributes → derived metric fields                                                                                                                                                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`http.server.request.duration`](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.0/docs/http/http-metrics.md) | `http.request.method` | `http.request.method` → `transactionName`<br> `http.route` → `transactionName` suffix (fallback: `unknown`)<br> `error.type` → `error.type`                                                                                                                      |
| `http.server.duration` _(legacy v1.20)_                                                                                         | `http.method`         | `http.method` → `transactionName`<br> `http.route` → `transactionName` suffix (fallback: `unknown`)<br> `http.status_code` → `error.type` if >= 500                                                                                                              |
| `rpc.server.duration`                                                                                                           | `rpc.system`          | `rpc.system` → `transactionName`<br> `rpc.service` → service segment of `transactionName` (fallback: `unknown`)<br> `rpc.method` → method segment of `transactionName` (fallback: `unknown`)<br> `rpc.grpc.status_code` → `error.type` if in `[2,4,12,13,14,15]` |

Transaction name formats:

- HTTP: `WebTransaction/server/{http.request.method} {http.route}`
- RPC: `WebTransaction/server/{rpc.system}/{rpc.service}.{rpc.method}`

> **Note:** Unlike New Relic language agents, OTel services do **not** produce `Transaction` events or transaction traces. The `transactionName` attribute exists only as an attribute on the `apm.service.transaction.duration` metric.

> **Seeing "unknown" in transaction names?** `http.route` is missing from your `http.server.request.duration` (v1.23) or `http.server.duration` (v1.20) metric.

---

### `apm.service.transaction.overview`

**Powers:** Transaction segment breakdown, "Time consumption by caller" on the Databases and  
External Services pages

**Always derived from spans, never from Source OTel metrics.** New Relic correlates client spans to  
their parent server span to build the breakdown.

```
Server span  (span.kind = server)
└─ DB client span    (span.kind = client, db.system present)
└─ HTTP client span  (span.kind = client, http.request.method present)
└─ RPC client span   (span.kind = client, rpc.system present)
```

#### Source inbound / root transaction spans

| Convention                                                                                                               | Span kind  | Required              | Source attributes → derived metric fields                                                                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------ | ---------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [HTTP v1.23](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.0/docs/http/http-metrics.md)              | `server`   | `http.request.method` | `http.request.method` → `transactionName`<br> `http.route` → `transactionName` suffix (fallback: `unknown`)                                                                                                                                       |
| HTTP v1.20 _(legacy)_                                                                                                    | `server`   | `http.method`         | `http.method` → `transactionName`<br> `http.route` → `transactionName` suffix (fallback: `unknown`)                                                                                                                                               |
| RPC v1.20                                                                                                                | `server`   | `rpc.system`          | `rpc.system` → `transactionName`<br> `rpc.service` → service segment of `transactionName` (fallback: `unknown`)<br> `rpc.method` → method segment of `transactionName` (fallback: `unknown`)                                                      |
| [Messaging v1.24](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/messaging/messaging-spans.md) | `consumer` | `messaging.operation` | `messaging.operation` → operation segment of `transactionName`<br> `messaging.destination.template` → destination segment of `transactionName` (preferred)<br> `messaging.destination.name` → destination segment of `transactionName` (fallback) |

Transaction name formats:

- HTTP: `WebTransaction/server/{http.request.method} {http.route}`
- RPC: `WebTransaction/server/{rpc.system}/{rpc.service}.{rpc.method}`
- Messaging: `OtherTransaction/consumer/{messaging.operation}/{messaging.destination.template}`

> **Note:** Unlike New Relic language agents, OTel services do **not** produce `Transaction` events or transaction traces. The `transactionName` attribute exists only as an attribute on the `apm.service.transaction.duration` metric.

#### Source outbound DB client spans

These contribute segment time to their parent transaction.

| Convention                                                                                                                  | Span kind              | Required              | Source attributes → derived metric fields                                                                                                                                                                                                                                                  |
| --------------------------------------------------------------------------------------------------------------------------- | ---------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [DB v1.33](https://github.com/open-telemetry/semantic-conventions/blob/v1.33.0/docs/database/database-metrics.md)           | `client` or `internal` | `db.system.name`      | `db.system.name` → `db.system`<br> `db.operation.name` → `db.operation`<br> `db.stored_procedure.name` → `db.sql.table` (highest priority)<br> `db.collection.name` → `db.sql.table` (fallback)<br> `db.query.summary` → parsed for both `db.sql.table` and `db.operation` (last fallback) |
| [Redis v1.24](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md)          | `client`               | `db.system = 'redis'` | `db.system` → `db.system`<br> `db.operation` → `db.operation` (fallback: span `name`)<br> `db.sql.table` → `db.sql.table`                                                                                                                                                                  |
| [DB v1.24 _(generic)_](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md) | `client`               | `db.system`           | `db.system` → `db.system`<br> `db.operation` → `db.operation`<br> `db.sql.table` → `db.sql.table`                                                                                                                                                                                          |

#### Source outbound HTTP/RPC client spans

These contribute segment time to their parent transaction.

| Convention                                                                                                         | Span kind | Required              | Source attributes → derived metric fields                                                          |
| ------------------------------------------------------------------------------------------------------------------ | --------- | --------------------- | -------------------------------------------------------------------------------------------------- |
| [HTTP client v1.23](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.0/docs/http/http-metrics.md) | `client`  | `http.request.method` | `http.request.method` → `http.method`<br> `server.address` → `external.host` (fallback: `unknown`) |
| HTTP client v1.20 _(legacy)_                                                                                       | `client`  | `http.method`         | `http.method` → `http.method`<br> `net.peer.name` → `external.host` (fallback: `unknown`)          |
| RPC client v1.20                                                                                                   | `client`  | `rpc.system`          | `rpc.system` → `rpc.system`<br> `net.peer.name` → `external.host` (fallback: `unknown`)            |

> Because this is span-based, your sampling strategy directly affects accuracy. If you only  
> sample errors, the segment breakdown will be skewed.

---

### `apm.service.external.host.duration`

**Powers:** External Services page

Synthesized from your **outbound HTTP or RPC client metrics**.

| Source OTel metric                                                                                                              | Required              | Source attributes → derived metric fields                                                          |
| ------------------------------------------------------------------------------------------------------------------------------- | --------------------- | -------------------------------------------------------------------------------------------------- |
| [`http.client.request.duration`](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.0/docs/http/http-metrics.md) | `http.request.method` | `http.request.method` → `http.method`<br> `server.address` → `external.host` (fallback: `unknown`) |
| `http.client.duration` _(legacy v1.20)_                                                                                         | `http.method`         | `http.method` → `http.method`<br> `net.peer.name` → `external.host` (fallback: `unknown`)          |
| `rpc.client.duration`                                                                                                           | `rpc.system`          | `rpc.system` → `rpc.system`<br> `net.peer.name` → `external.host` (fallback: `unknown`)            |

---

### `apm.service.datastore.operation.duration`

**Powers:** Databases page

New Relic uses the presence of `db.system` vs `db.system.name` on a span to determine which convention the instrumentation is using, and chooses accordingly:

- **`db.system` present** → [old convention (v1.24)](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md); no metric data is expected, so New Relic derives this metric directly from the **span**.
- **`db.system.name` present** → [new stable convention (v1.33)](https://github.com/open-telemetry/semantic-conventions/blob/v1.33.0/docs/database/database-metrics.md); New Relic uses the `db.client.operation.duration` **metric**.

| Convention                                                                                                                                                           | OTel source                                                       | Required              | Source attributes → derived metric fields                                                                                                                                                                                                                                                  |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [DB span convention v1.24 _(legacy)_](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md)                           | Span (`db.system` present)                                        | `db.system`           | `db.system` → `db.system`<br> `db.operation` → `db.operation` (fallback: `unknown`) **FACT CHECK THIS**<br> `db.sql.table` → `db.sql.table` **FACT CHECK THIS**                                                                                                                            |
| **FACT CHECK THIS**<br>[Redis span convention v1.24 _(legacy)_](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md) | Span (`db.system = 'redis'`)                                      | `db.system = 'redis'` | `db.system` → `db.system`<br> `db.operation` → `db.operation` (fallback: span `name`)<br> `db.sql.table` → `db.sql.table`                                                                                                                                                                  |
| [DB metrics convention v1.33](https://github.com/open-telemetry/semantic-conventions/blob/v1.33.0/docs/database/database-metrics.md)                                 | Metric: `db.client.operation.duration` (`db.system.name` present) | `db.system.name`      | `db.system.name` → `db.system`<br> `db.operation.name` → `db.operation`<br> `db.stored_procedure.name` → `db.sql.table` (highest priority)<br> `db.collection.name` → `db.sql.table` (fallback)<br> `db.query.summary` → parsed for both `db.sql.table` and `db.operation` (last fallback) |

Most database instrumentation has not yet adopted v1.33 stable conventions, so the span-based path is the common case today. As stable instrumentation becomes available for your language, upgrading your DB instrumentation library to a version that emits `db.client.operation.duration` will shift this to the more accurate metrics-based path.

---

## Special cases

### Messaging / non-web transactions

Messaging conventions are still in development — **no stable messaging metrics exist yet**. Non-web transactions for message consumers are always derived from consumer spans with `messaging.operation` set. Throughput accuracy depends on your sampling rate.

### Ruby

OpenTelemetry Ruby does not yet emit metrics. All APM metrics for Ruby services are synthesized from spans.

### Any span-derived metric

If a metric falls back to span data, throughput numbers reflect your sample rate, not real  
traffic volume. A 10% sample rate means throughput appears ~10x lower than reality.

---

## What you need to instrument, in plain terms

**Summary + Transactions pages:**
Emit either `http.server.request.duration` with `http.request.method` and `http.route`, _or_ `rpc.server.duration` with `rpc.system`, `rpc.service`, and `rpc.method`.

**Databases page:**
Emit `db.client.operation.duration` (v1.33) with `db.system.name`. If your DB instrumentation library doesn't yet emit `db.client.operation.duration`, DB client spans with `db.system` are used as a fallback.

**External Services page:**
Emit either `http.client.request.duration` with `http.request.method` and `server.address`, _or_ `rpc.client.duration` with `rpc.system` and `net.peer.name`.

**Segment breakdowns (inside a transaction):**
Ensure spans are emitted with `span.kind = server` on inbound request spans and `span.kind = client` on outbound DB/HTTP/RPC spans, with the relevant semantic convention attributes listed in the `apm.service.transaction.overview` tables above. Segment breakdowns always come from spans.

---

## Troubleshooting

### Step 1 — Is any data reaching New Relic?

```
(1) Is the service entity visible in New Relic at all?
            │
     NO ────┴──► Data is not reaching New Relic.
                 Check your OTel exporter endpoint and API key.
        YES │
            ▼
(2) Have apm.service.* metrics been synthesized?
            │
     NO ────┴──► Raw data is arriving but synthesis has not occurred.
                 Check that your metrics and spans include service.name.
        YES │
            ▼
  Use the symptom-specific trees below for the affected page. ✓
```

**Verification queries:**

```sql
-- (1) Check raw OTel data is arriving (metrics and spans)
FROM Metric, Span SELECT count(*) WHERE service.name = 'SERVICE_NAME' SINCE 1 day ago

-- (2) Check whether apm.service.* metrics have been synthesized at all
FROM Metric SELECT count(*) WHERE metricName LIKE 'apm.service.%' AND service.name = 'SERVICE_NAME' FACET metricName SINCE 1 day ago
```

---

### Summary / Transactions page is blank

```
(1) Are you emitting a server-side duration metric?
  • http.server.request.duration  (HTTP v1.23)
  • http.server.duration          (HTTP v1.20 legacy)
  • rpc.server.duration           (RPC)
            │
     NO ────┴──► No recognized server metric found.
            │     Emit one of the above from your HTTP or RPC server instrumentation.
        YES │
            ▼
(2) Does the metric include the required attribute?
  • http.server.request.duration → http.request.method must be present
  • http.server.duration         → http.method must be present
  • rpc.server.duration          → rpc.system must be present
            │
     NO ────┴──► Add the missing required attribute to the metric indicated above.
            │
       YES  │
            ▼
  Page should be populated. ✓

(3) Transaction names showing "unknown"?
  └──► HTTP: http.route is missing.
       Add http.route to your http.server.request.duration metric (v1.23)
       or http.server.duration metric (v1.20 legacy).
  └──► RPC: rpc.service or rpc.method is missing.
       Add them to your rpc.server.duration metric.

(4) Verify the synthesized metric exists and inspect transaction names.
```

**Verification queries:**

```sql
-- (1) Check which server metrics are arriving
FROM Metric SELECT count(*) WHERE metricName IN ('http.server.request.duration', 'http.server.duration', 'rpc.server.duration') AND service.name = 'SERVICE_NAME' FACET metricName SINCE 1 day ago

-- (2) Check required attributes are present on each metric
FROM Metric SELECT count(*) WHERE metricName = 'http.server.request.duration' AND service.name = 'SERVICE_NAME' FACET http.request.method SINCE 1 day ago

-- (3) Check http.route is present (missing values here = "unknown" transaction names)
FROM Metric SELECT count(*) WHERE metricName = 'http.server.request.duration' AND service.name = 'SERVICE_NAME' FACET http.route SINCE 1 day ago

-- (4) Verify synthesized apm.service.transaction.duration exists and see transaction names
FROM Metric SELECT count(*) WHERE metricName = 'apm.service.transaction.duration' AND service.name = 'SERVICE_NAME' FACET transactionName SINCE 1 day ago
```

---

### Databases page is blank

```
(1) Are you using Ruby?
            │
    YES ────┴──► Ruby emits no metrics. DB data comes from spans only.
            │    Ensure DB client spans are emitted with db.system present.
            │    Skip to step (3).
        NO  │
            ▼
(2) Do your DB spans have db.system.name?  (v1.33 stable convention)
            │
    YES ────┴──► (3) Are you also emitting the db.client.operation.duration metric?
            │         NO  → Your DB instrumentation library emits spans but not the
            │               db.client.operation.duration metric yet.
            │               Check for a newer version of your DB instrumentation library.
            │         YES → Databases page should be populated. ✓
        NO  │
            ▼
(3) Do your DB spans have db.system?  (v1.24 legacy convention)
            │
     NO ────┴──► DB spans are missing both db.system and db.system.name.
            │    Check your DB instrumentation library.
        YES │
            ▼
  New Relic will derive the metric from spans. ✓

  Accuracy depends on your sampling rate — see "Any span-derived metric" above.

(4) Operations or table names showing as "unknown"?
  v1.33 (db.system.name) — source is the db.client.operation.duration metric:
  └──► db.operation showing "unknown" → add db.operation.name to the db.client.operation.duration metric,
       or add db.query.summary (New Relic parses the operation from the first word of the summary)
  └──► db.sql.table missing → add db.stored_procedure.name to the db.client.operation.duration metric,
       or db.collection.name (fallback), or db.query.summary (New Relic parses the table name
       from the non-keyword words in the summary)

  v1.24 legacy (db.system) — source is DB client spans:
  └──► db.operation showing "unknown" → add db.operation attribute to your DB client spans
  └──► db.sql.table missing → add db.sql.table attribute to your DB client spans

(5) Verify the synthesized metric exists and inspect operation and table values.
```

**Verification queries:**

```sql
-- (2) Check for v1.33 DB spans (db.system.name present)
FROM Span SELECT count(*) WHERE db.system.name IS NOT NULL AND service.name = 'SERVICE_NAME' FACET db.system.name SINCE 1 day ago

-- (3a) Check for v1.33 db.client.operation.duration metric
FROM Metric SELECT count(*) WHERE metricName = 'db.client.operation.duration' AND service.name = 'SERVICE_NAME' SINCE 1 day ago

-- (3b) Check for v1.24 legacy DB spans (db.system present, no db.system.name)
FROM Span SELECT count(*) WHERE db.system IS NOT NULL AND db.system.name IS NULL AND service.name = 'SERVICE_NAME' FACET db.system SINCE 1 day ago

-- (4) Check for unknown operations or tables on db.client.operation.duration (v1.33)
FROM Metric SELECT count(*) WHERE metricName = 'db.client.operation.duration' AND service.name = 'SERVICE_NAME' FACET db.operation.name, db.query.summary SINCE 1 day ago

-- (5) Verify synthesized apm.service.datastore.operation.duration exists
FROM Metric SELECT count(*) WHERE metricName = 'apm.service.datastore.operation.duration' AND service.name = 'SERVICE_NAME' FACET db.system, db.operation SINCE 1 day ago
```

---

### External Services page is blank

```
(1) Are you emitting a client-side duration metric?
  • http.client.request.duration  (HTTP v1.23)
  • http.client.duration          (HTTP v1.20 legacy)
  • rpc.client.duration           (RPC)
            │
     NO ────┴──► No recognized client metric found.
            │     Emit one of the above from your outbound HTTP or RPC client instrumentation.
        YES │
            ▼
(2) Does the metric include the required attribute?
  • http.client.request.duration → http.request.method must be present
  • http.client.duration         → http.method must be present
  • rpc.client.duration          → rpc.system must be present
            │
     NO ────┴──► Add the missing required attribute to the metric indicated above.
            │
        YES │
            ▼
  Page should be populated. ✓

(3) All hosts showing as "unknown"?
  └──► Add server.address to your http.client.request.duration metric (v1.23),
       or net.peer.name to your http.client.duration or rpc.client.duration metric (v1.20 legacy).

(4) Verify the synthesized metric exists and inspect host values.
```

**Verification queries:**

```sql
-- (1) Check which client metrics are arriving
FROM Metric SELECT count(*) WHERE metricName IN ('http.client.request.duration', 'http.client.duration', 'rpc.client.duration') AND service.name = 'SERVICE_NAME' FACET metricName SINCE 1 day ago

-- (2) Check required attributes are present on each metric
FROM Metric SELECT count(*) WHERE metricName = 'http.client.request.duration' AND service.name = 'SERVICE_NAME' FACET http.request.method SINCE 1 day ago

-- (3a) Check server.address on http.client.request.duration (missing = "unknown" hosts)
FROM Metric SELECT count(*) WHERE metricName = 'http.client.request.duration' AND service.name = 'SERVICE_NAME' FACET server.address SINCE 1 day ago

-- (3b) Check net.peer.name on http.client.duration or rpc.client.duration (missing = "unknown" hosts)
FROM Metric SELECT count(*) WHERE metricName IN ('http.client.duration', 'rpc.client.duration') AND service.name = 'SERVICE_NAME' FACET net.peer.name SINCE 1 day ago

-- (4) Verify synthesized apm.service.external.host.duration exists
FROM Metric SELECT count(*) WHERE metricName = 'apm.service.external.host.duration' AND service.name = 'SERVICE_NAME' FACET external.host SINCE 1 day ago
```

---

### Transaction segment breakdown is blank or inaccurate

```
(1) Are spans being emitted at all?
            │
     NO ────┴──► Enable span export in your OTel SDK configuration.
            │
        YES │
            ▼
(2) Do spans have the correct span.kind?
  • Inbound HTTP/RPC requests → span.kind = server
  • Inbound messaging consumers → span.kind = consumer
  • Outbound DB / HTTP / RPC calls → span.kind = client
            │
     NO ────┴──► span.kind is missing or incorrect on your spans.
            │     Check your instrumentation library — span.kind is usually set automatically
            │     by OTel instrumentation libraries, not manually.
        YES │
            ▼
(3) Are client spans children of the server span in the same trace?
(i.e. DB/HTTP/RPC client spans are nested under the inbound server span)
            │
     NO ────┴──► Context propagation may be broken.
            │     Ensure your OTel SDK has context propagation configured and that
            │     downstream calls inherit the active span context.
        YES │
            ▼
  Breakdown should be visible. ✓

  Numbers look much lower than expected?
  └──► Segment breakdown is extrapolated from sampled spans.
       If you sample only errors, results will be skewed.
       See "Any span-derived metric" in Special cases above.
```

**Verification queries:**

```sql
-- (1) Check spans are arriving
FROM Span SELECT count(*) WHERE service.name = 'SERVICE_NAME' SINCE 1 day ago

-- (2) Check span.kind distribution (should show server, client, and/or consumer)
FROM Span SELECT count(*) WHERE service.name = 'SERVICE_NAME' FACET span.kind SINCE 1 day ago

-- (3) Check that client spans have a parent span (confirms context propagation is working)
FROM Span SELECT count(*) WHERE span.kind = 'client' AND service.name = 'SERVICE_NAME' FACET parentId IS NOT NULL SINCE 1 day ago
```

---

### Error rate is zero / Errors Inbox is empty

```
Error rate on Summary / Transactions page is zero:
(1) Are error attributes present on the correct metric?
  • http.server.request.duration → add error.type attribute on error requests
  • http.server.duration         → add http.status_code attribute (errors counted when >= 500)
  • rpc.server.duration          → add rpc.grpc.status_code attribute (errors counted
                                   when value is in [2, 4, 12, 13, 14, 15])

(2) Verify the synthesized error count metric exists.

Errors Inbox is empty:
(3) Are failed spans setting otel.status_code = ERROR?
  └──► Span exception events alone do NOT mark a span as an error.
       You must explicitly set span status to ERROR on the span itself.
```

**Verification queries:**

```sql
-- (1a) Check error.type is present on http.server.request.duration
FROM Metric SELECT count(*) WHERE metricName = 'http.server.request.duration' AND service.name = 'SERVICE_NAME' FACET error.type SINCE 1 day ago

-- (1b) Check http.status_code on http.server.duration (errors counted when >= 500)
FROM Metric SELECT count(*) WHERE metricName = 'http.server.duration' AND service.name = 'SERVICE_NAME' FACET http.status_code SINCE 1 day ago

-- (1c) Check rpc.grpc.status_code on rpc.server.duration (errors counted when in [2,4,12,13,14,15])
FROM Metric SELECT count(*) WHERE metricName = 'rpc.server.duration' AND service.name = 'SERVICE_NAME' FACET rpc.grpc.status_code SINCE 1 day ago

-- (2) Verify synthesized apm.service.error.count exists
FROM Metric SELECT count(*) WHERE metricName = 'apm.service.error.count' AND service.name = 'SERVICE_NAME' FACET transactionName SINCE 1 day ago

-- (3) Check error spans are being emitted with otel.status_code = ERROR
FROM Span SELECT count(*) WHERE otel.status_code = 'ERROR' AND service.name = 'SERVICE_NAME' SINCE 1 day ago
```
