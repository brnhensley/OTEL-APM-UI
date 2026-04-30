**Note**: this is a work in progress, please help fact check and let me know how to improve.

# OpenTelemetry APM UI — What Data Powers What

New Relic does not display raw OpenTelemetry data directly in the APM UI. Instead, it **synthesizes a set of `apm.service.*` metrics** from OTel data, then uses those to drive the UI — the same way the New Relic language agents do. The original OTel data is still available for custom queries.

## Contents

- [How it works](#how-it-works-the-short-version)
- [Quick reference](#quick-reference-what-does-each-ui-section-need)
- [Metric-by-metric breakdown](#metric-by-metric-breakdown)
  - [`apm.service.transaction.duration`](#apmservicetransactionduration)
  - [`apm.service.transaction.overview`](#apmservicetransactionoverview)
  - [`apm.service.external.host.duration`](#apmserviceexternalhostduration)
  - [`apm.service.datastore.operation.duration`](#apmservicedatastoreoperationduration)
- [Special cases](#special-cases)
- [What data is needed](#what-data-is-needed)
- [Troubleshooting Flowcharts](#troubleshooting)

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

| UI Section                                | Powered by                                                                          | Required OTel source                                 |
| ----------------------------------------- | ----------------------------------------------------------------------------------- | ---------------------------------------------------- |
| **Summary**                               | [`apm.service.transaction.duration`](#apmservicetransactionduration)                | HTTP or RPC server metrics; messaging consumer spans |
| **Transactions**                          | [`apm.service.transaction.duration`](#apmservicetransactionduration)                | HTTP or RPC server metrics; messaging consumer spans |
| **Transaction segment breakdown**         | [`apm.service.transaction.overview`](#apmservicetransactionoverview)                | Spans only — server + nested client spans            |
| **Databases**                             | [`apm.service.datastore.operation.duration`](#apmservicedatastoreoperationduration) | DB metrics (v1.33) or DB client spans (v1.24 legacy) |
| **DB "Time consumption by caller"**       | [`apm.service.transaction.overview`](#apmservicetransactionoverview)                | Spans only                                           |
| **External Services**                     | [`apm.service.external.host.duration`](#apmserviceexternalhostduration)             | HTTP/RPC client metrics                              |
| **External "Time consumption by caller"** | [`apm.service.transaction.overview`](#apmservicetransactionoverview)                | Spans only                                           |
| **Distributed Tracing**                   | —                                                                                   | Spans                                                |
| **Errors Inbox**                          | —                                                                                   | Spans with `otel.status_code = ERROR`                |
| **Logs**                                  | —                                                                                   | Logs                                                 |
| **JVM Runtime**                           | —                                                                                   | JVM runtime metrics (OTel Java agent)                |
| **Go Runtime**                            | —                                                                                   | Go runtime metrics (OTel Go contrib)                 |

---

## Metric-by-metric breakdown

### `apm.service.transaction.duration`

**Powers:** Summary page, Transactions page, error rate

Synthesized from one of these **server-side HTTP or RPC metrics**.

| Source OTel metric                                                                                                                  | Required              | Source attributes → derived metric fields                                                                                                                                                                                                                        |
| ----------------------------------------------------------------------------------------------------------------------------------- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`http.server.request.duration`](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.0/docs/http/http-metrics.md)     | `http.request.method` | `http.request.method` → `transactionName`<br> `http.route` → `transactionName` suffix (fallback: `unknown`)<br> `error.type` → `error.type`                                                                                                                      |
| `http.server.duration` _(legacy v1.20)_                                                                                             | `http.method`         | `http.method` → `transactionName`<br> `http.route` → `transactionName` suffix (fallback: `unknown`)<br> `http.status_code` → `error.type` if >= 500                                                                                                              |
| [`rpc.server.call.duration`](https://github.com/open-telemetry/semantic-conventions/blob/v1.40.0/docs/rpc/rpc-metrics.md) _(v1.40)_ | `rpc.system.name`     | `rpc.system.name` → `transactionName`<br> `rpc.method` → method segment of `transactionName` (fallback: `unknown`)<br> `error.type` → `error.type`                                                                                                               |
| `rpc.server.duration` _(legacy v1.20)_                                                                                              | `rpc.system`          | `rpc.system` → `transactionName`<br> `rpc.service` → service segment of `transactionName` (fallback: `unknown`)<br> `rpc.method` → method segment of `transactionName` (fallback: `unknown`)<br> `rpc.grpc.status_code` → `error.type` if in `[2,4,12,13,14,15]` |

Transaction name formats:

- HTTP: `WebTransaction/server/{http.request.method} {http.route}`
- RPC v1.40: `WebTransaction/server/{rpc.system.name}/{rpc.method}` _(no service segment)_
- RPC v1.20 _(legacy)_: `WebTransaction/server/{rpc.system}/{rpc.service}.{rpc.method}`

> **Note:** Unlike New Relic language agents, OTel services do **not** produce `Transaction` events or transaction traces. The `transactionName` attribute exists only as an attribute on the `apm.service.transaction.duration` metric.

> **Seeing "unknown" segments in transaction names?**
>
> - HTTP: `http.route` is missing from the `http.server.request.duration` (v1.23) or `http.server.duration` (v1.20) metric.
> - RPC v1.40: `rpc.method` is missing from the `rpc.server.call.duration` metric.
> - RPC v1.20: `rpc.service` or `rpc.method` is missing from the `rpc.server.duration` metric.

> **Seeing `WebTransaction/server/unknown`? (FallbackServer)** A recognized server metric arrived but the required identifying attribute was null or missing — `http.request.method` for `http.server.request.duration`, `http.method` for `http.server.duration`, `rpc.system.name` for `rpc.server.call.duration`, or `rpc.system` for `rpc.server.duration`. When that happens, a FallbackServer rule fires and still produces the metric, but the transaction name is hardcoded to `WebTransaction/server/unknown`. The fix is to ensure the required attribute is present on the metric.

> **Messaging consumer spans also produce `apm.service.transaction.duration`** via the OtelMessagingConsumer1_24 and FallbackConsumer rules. See [Messaging / non-web transactions](#messaging--non-web-transactions).

---

### `apm.service.transaction.overview`

**Powers:** Transaction segment breakdown, "Time consumption by caller" on the Databases and  
External Services pages

**Always derived from spans, never from Source OTel metrics.** New Relic correlates client spans to  
their parent server span to build the breakdown.

```
Server span    (span.kind = server)
└─ DB client span    (span.kind = client, db.system.name or db.system present)
└─ HTTP client span  (span.kind = client, http.request.method or http.method present)
└─ RPC client span   (span.kind = client, rpc.system.name or rpc.system present)

Consumer span  (span.kind = consumer)
└─ DB client span    (span.kind = client, db.system.name or db.system present)
└─ HTTP client span  (span.kind = client, http.request.method or http.method present)
└─ RPC client span   (span.kind = client, rpc.system.name or rpc.system present)
```

#### Source inbound / root transaction spans

| Convention                                                                                                               | `span.kind` | Required              | Source attributes → derived metric fields                                                                                                                                                                                                         |
| ------------------------------------------------------------------------------------------------------------------------ | ----------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [HTTP v1.23](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.0/docs/http/http-metrics.md)              | `server`    | `http.request.method` | `http.request.method` → `transactionName`<br> `http.route` → `transactionName` suffix (fallback: `unknown`)                                                                                                                                       |
| HTTP v1.20 _(legacy)_                                                                                                    | `server`    | `http.method`         | `http.method` → `transactionName`<br> `http.route` → `transactionName` suffix (fallback: `unknown`)                                                                                                                                               |
| [RPC v1.40](https://github.com/open-telemetry/semantic-conventions/blob/v1.40.0/docs/rpc/rpc-metrics.md)                 | `server`    | `rpc.system.name`     | `rpc.system.name` → `transactionName`<br> `rpc.method` → method segment of `transactionName` (fallback: `unknown`)                                                                                                                                |
| RPC v1.20 _(legacy)_                                                                                                     | `server`    | `rpc.system`          | `rpc.system` → `transactionName`<br> `rpc.service` → service segment of `transactionName` (fallback: `unknown`)<br> `rpc.method` → method segment of `transactionName` (fallback: `unknown`)                                                      |
| [Messaging v1.24](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/messaging/messaging-spans.md) | `consumer`  | `messaging.operation` | `messaging.operation` → operation segment of `transactionName`<br> `messaging.destination.template` → destination segment of `transactionName` (preferred)<br> `messaging.destination.name` → destination segment of `transactionName` (fallback) |

Transaction name formats:

- HTTP: `WebTransaction/server/{http.request.method} {http.route}`
- RPC v1.40: `WebTransaction/server/{rpc.system.name}/{rpc.method}` _(no service segment)_
- RPC v1.20 _(legacy)_: `WebTransaction/server/{rpc.system}/{rpc.service}.{rpc.method}`
- Messaging one of these
  - `OtherTransaction/consumer/{messaging.operation}/{messaging.destination.template}`
  - `OtherTransaction/consumer/{messaging.operation}/{messaging.destination.name}`
  - `OtherTransaction/consumer/unknown` _(FallbackConsumer — see note below)_

> **Note:** `OtherTransaction/consumer/unknown` means `messaging.operation` was null on the consumer span. In this case, synthesis falls back to a FallbackConsumer rule that still produces a metric — so despite what the "Required" column above suggests, `messaging.operation` is not strictly required. Consumer spans without it will still generate a metric, just with `unknown` as the operation segment.

#### Source outbound client spans

These contribute segment time to their parent transaction.

| Convention                                                                                                                  | `span.kind`            | Required              | Source attributes → derived metric fields                                                                                                                                                                                                                                                  |
| --------------------------------------------------------------------------------------------------------------------------- | ---------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [DB v1.33](https://github.com/open-telemetry/semantic-conventions/blob/v1.33.0/docs/database/database-metrics.md)           | `client` or `internal` | `db.system.name`      | `db.system.name` → `db.system`<br> `db.operation.name` → `db.operation`<br> `db.stored_procedure.name` → `db.sql.table` (highest priority)<br> `db.collection.name` → `db.sql.table` (fallback)<br> `db.query.summary` → parsed for both `db.sql.table` and `db.operation` (last fallback) |
| [Redis v1.24](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md)          | `client`               | `db.system = 'redis'` | `db.system` → `db.system`<br> `db.operation` → `db.operation` (fallback: span `name`)<br> `db.sql.table` → `db.sql.table`                                                                                                                                                                  |
| [DB v1.24 _(generic)_](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md) | `client`               | `db.system`           | `db.system` → `db.system`<br> `db.operation` → `db.operation` (fallback: `unknown`)<br> `db.sql.table` → `db.sql.table`                                                                                                                                                                    |
| [HTTP client v1.23](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.0/docs/http/http-metrics.md)          | `client`               | `http.request.method` | `server.address` → `external.host` (fallback: `unknown`)                                                                                                                                                                                                                                   |
| HTTP client v1.20 _(legacy)_                                                                                                | `client`               | `http.method`         | `net.peer.name` → `external.host` (fallback: `unknown`)                                                                                                                                                                                                                                    |
| [RPC client v1.40](https://github.com/open-telemetry/semantic-conventions/blob/v1.40.0/docs/rpc/rpc-metrics.md)             | `client`               | `rpc.system.name`     | `server.address` → `external.host` (fallback: `unknown`)                                                                                                                                                                                                                                   |
| RPC client v1.20 _(legacy)_                                                                                                 | `client`               | `rpc.system`          | `net.peer.name` → `external.host` (fallback: `server.address`, then `unknown`)                                                                                                                                                                                                             |

> Because this is span-based, sampling strategy directly affects accuracy. If only errors are sampled, the segment breakdown will be skewed.

---

### `apm.service.external.host.duration`

**Powers:** External Services page

Synthesized from one of these **outbound HTTP or RPC client metrics** (matched by `span.kind = client` on the source span or metric).

| Source OTel metric                                                                                                                  | Required              | Source attributes → derived metric fields                                      |
| ----------------------------------------------------------------------------------------------------------------------------------- | --------------------- | ------------------------------------------------------------------------------ |
| [`http.client.request.duration`](https://github.com/open-telemetry/semantic-conventions/blob/v1.23.0/docs/http/http-metrics.md)     | `http.request.method` | `server.address` → `external.host` (fallback: `unknown`)                       |
| `http.client.duration` _(legacy v1.20)_                                                                                             | `http.method`         | `net.peer.name` → `external.host` (fallback: `unknown`)                        |
| [`rpc.client.call.duration`](https://github.com/open-telemetry/semantic-conventions/blob/v1.40.0/docs/rpc/rpc-metrics.md) _(v1.40)_ | `rpc.system.name`     | `server.address` → `external.host` (fallback: `unknown`)                       |
| `rpc.client.duration` _(legacy v1.20)_                                                                                              | `rpc.system`          | `net.peer.name` → `external.host` (fallback: `server.address`, then `unknown`) |

> **External Services showing only `unknown` hosts? (FallbackClient)** A recognized client metric arrived but the required identifying attribute was null or missing — `http.request.method` for `http.client.request.duration`, `http.method` for `http.client.duration`, `rpc.system.name` for `rpc.client.call.duration`, or `rpc.system` for `rpc.client.duration`. A FallbackClient rule fires and still produces the metric, but `external.host` is hardcoded to `unknown`, so all external calls collapse into one entry. The fix is to ensure the required attribute is present on the metric.

---

### `apm.service.datastore.operation.duration`

**Powers:** Databases page

These rules match **different input data types** and are independent — a service emitting both metrics and spans can have multiple rules fire simultaneously:

- **`db.client.operation.duration` metric with `db.system.name`** → [v1.33 stable convention](https://github.com/open-telemetry/semantic-conventions/blob/v1.33.0/docs/database/database-metrics.md); synthesized from the metric.
- **DB client span with `db.system = 'redis'`** → [Redis span convention (v1.24)](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md); synthesized from the span, with span `name` as operation fallback.
- **DB client span with `db.system`** (any other value) → [generic span convention (v1.24)](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md); synthesized from the span.

| Convention                                                                                                                                         | OTel source                                                       | Required              | Source attributes → derived metric fields                                                                                                                                                                                                                                                                                                                                                                         |
| -------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- | --------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [DB span convention v1.24 _(generic legacy)_](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md) | Span (`db.system` present, not `redis`)                           | `db.system`           | `db.system` → `db.system`<br> `db.operation` → `db.operation` (fallback: `unknown`)<br> `db.sql.table` → `db.sql.table` (no fallback — may be absent)                                                                                                                                                                                                                                                             |
| [Redis span convention v1.24 _(legacy)_](https://github.com/open-telemetry/semantic-conventions/blob/v1.24.0/docs/database/database-spans.md)      | Span (`db.system = 'redis'`)                                      | `db.system = 'redis'` | `db.system` → `db.system`<br> `db.operation` → `db.operation` (fallback: span `name`, then `unknown`)<br> `db.sql.table` → `db.sql.table` (no fallback — may be absent)                                                                                                                                                                                                                                           |
| [DB metrics convention v1.33](https://github.com/open-telemetry/semantic-conventions/blob/v1.33.0/docs/database/database-metrics.md)               | Metric: `db.client.operation.duration` (`db.system.name` present) | `db.system.name`      | `db.system.name` → `db.system`<br> `db.operation.name` → `db.operation` (fallback: first word of `db.query.summary`, then `unknown`)<br> `db.stored_procedure.name` → `db.sql.table` (highest priority)<br> `db.collection.name` → `db.sql.table` (fallback)<br> first non-keyword word of `db.query.summary` → `db.sql.table` (last fallback)<br> `db.query.summary` → `db.query.summary` (passed through as-is) |

Most database instrumentation has not yet adopted v1.33 stable conventions, so the span-based path is the common case today. As stable instrumentation becomes available, upgrading the DB instrumentation library to a version that emits `db.client.operation.duration` will shift this to the more accurate metrics-based path.

---

## Special cases

### Messaging / non-web transactions

Messaging conventions are still in development — **no stable messaging metrics exist yet**. Consumer transactions are derived from spans only (`span.kind = consumer`), so throughput accuracy depends on the sampling rate. See the FallbackConsumer note in [`apm.service.transaction.overview`](#apmservicetransactionoverview) for how `messaging.operation` behaves when absent.

### Ruby

OpenTelemetry Ruby does not yet emit metrics. All APM metrics for Ruby services are synthesized from spans.

### Any span-derived metric

If a metric falls back to span data, throughput numbers reflect the sample rate, not real traffic volume. A 10% sample rate means throughput appears ~10x lower than reality.

---

## What data is needed

**Summary + Transactions pages:**
Emit one of:

- `http.server.request.duration` with `http.request.method` and `http.route`
- `rpc.server.call.duration` (v1.40) with `rpc.system.name` and `rpc.method`
- `rpc.server.duration` (legacy v1.20) with `rpc.system`, `rpc.service`, and `rpc.method`

**Databases page:**
Emit one of:

- `db.client.operation.duration` metric (v1.33) with `db.system.name` — uses the metrics-based path.
- DB client spans with `db.system` (v1.24 legacy) — uses the span-based path directly (not a fallback for v1.33; this is a separate convention).

**External Services page:**
Emit one of:

- `http.client.request.duration` with `http.request.method` and `server.address`
- `rpc.client.call.duration` (v1.40) with `rpc.system.name` and `server.address`
- `rpc.client.duration` (legacy v1.20) with `rpc.system` and `net.peer.name` (or `server.address`)

**Segment breakdowns (inside a transaction):**
Ensure spans are emitted with `span.kind = server` on inbound request spans and `span.kind = client` on outbound DB/HTTP/RPC spans, with the relevant semantic convention attributes listed in the [`apm.service.transaction.overview`](#apmservicetransactionoverview) tables above. Segment breakdowns always come from spans.

**Error rate:**
The server metric must include an error attribute:

- `http.server.request.duration` with `error.type` on error requests
- `http.server.duration` with `http.status_code` (>= 500 counts as an error)
- `rpc.server.call.duration` with `error.type` on error requests
- `rpc.server.duration` with `rpc.grpc.status_code` (codes 2, 4, 12, 13, 14, 15 count as errors)

**Distributed Tracing:**
Emit spans with `service.name`.

**Errors Inbox:**
Emit spans with `otel.status_code = ERROR`.

---

## Troubleshooting

### Step 1 — Is any data reaching New Relic?

```
(1) Is the service entity visible in New Relic at all?
            │
     NO ────┴──► Data is not reaching New Relic.
                 Check the OTel exporter endpoint and API key.
        YES │
            ▼
(2) Have apm.service.* metrics been synthesized?
            │
     NO ────┴──► Raw data is arriving but synthesis has not occurred.
                 Check that metrics and spans include service.name.
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
(1) Is a server-side duration metric being emitted?
  • http.server.request.duration  (HTTP v1.23)
  • http.server.duration          (HTTP v1.20 legacy)
  • rpc.server.call.duration      (RPC v1.40)
  • rpc.server.duration           (RPC v1.20 legacy)
            │
     NO ────┴──► No recognized server metric found.
            │     Emit one of the above from the HTTP or RPC server instrumentation.
        YES │
            ▼
(2) Does the metric include the required attribute?
  • http.server.request.duration  → http.request.method must be present
  • http.server.duration          → http.method must be present
  • rpc.server.call.duration      → rpc.system.name must be present
  • rpc.server.duration           → rpc.system must be present
            │
     NO ────┴──► Add the missing required attribute to the metric indicated above.
            │    If it's present but still failing, a FallbackServer rule may match first and
            │    produce transactionName = WebTransaction/server/unknown. Check query (4).
            │
       YES  │
            ▼
  Page should be populated. ✓

(3) Transaction names showing "unknown"?
  └──► HTTP: http.route is missing.
       Add http.route to the http.server.request.duration metric (v1.23)
       or http.server.duration metric (legacy v1.20).
  └──► RPC (v1.40): rpc.method is missing. Add it to the rpc.server.call.duration metric.
  └──► RPC (v1.20): rpc.service or rpc.method is missing.
       Add them to the rpc.server.duration metric.

(4) Verify the synthesized metric exists and inspect transaction names.
```

**Verification queries:**

```sql
-- (1) Check which server metrics are arriving
FROM Metric SELECT count(*) WHERE metricName IN ('http.server.request.duration', 'http.server.duration', 'rpc.server.call.duration', 'rpc.server.duration') AND service.name = 'SERVICE_NAME' FACET metricName SINCE 1 day ago

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
(1) Is this a Ruby service?
            │
    YES ────┴──► Ruby emits no metrics. DB data comes from spans only.
            │    Ensure DB client spans are emitted with db.system present.
            │    Skip to step (3).
        NO  │
            ▼
(2) Do DB spans have db.system.name?  (v1.33 stable convention)
            │
    YES ────┴──► (3) Is db.client.operation.duration also being emitted?
            │         NO  → The DB instrumentation library emits spans but not the
            │               db.client.operation.duration metric yet.
            │               Check for a newer version of the DB instrumentation library.
            │         YES → Databases page should be populated. ✓
        NO  │
            ▼
(3) Do DB spans have db.system?  (v1.24 legacy convention)
            │
     NO ────┴──► DB spans are missing both db.system and db.system.name.
            │    Check the DB instrumentation library.
        YES │
            ▼
  New Relic will derive the metric from spans. ✓

  Accuracy depends on the sampling rate — see "Any span-derived metric" above.

(4) Operations or table names showing as "unknown"?
  v1.33 (db.system.name) — source is the db.client.operation.duration metric:
  └──► db.operation showing "unknown" → add db.operation.name to the db.client.operation.duration metric,
       or add db.query.summary (New Relic parses the operation from the first word of the summary)
  └──► db.sql.table missing → add db.stored_procedure.name to the db.client.operation.duration metric,
       or db.collection.name (fallback), or db.query.summary (New Relic parses the table name
       from the non-keyword words in the summary)

  v1.24 legacy (db.system) — source is DB client spans:
  └──► db.operation showing "unknown" → add db.operation attribute to DB client spans
  └──► db.sql.table missing → add db.sql.table attribute to DB client spans

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
(1) Is a client-side duration metric being emitted?
  • http.client.request.duration  (HTTP v1.23)
  • http.client.duration          (HTTP v1.20 legacy)
  • rpc.client.call.duration      (RPC v1.40)
  • rpc.client.duration           (RPC v1.20 legacy)
            │
     NO ────┴──► No recognized client metric found.
            │     Emit one of the above from the outbound HTTP or RPC client instrumentation.
        YES │
            ▼
(2) Does the metric include the required attribute?
  • http.client.request.duration  → http.request.method must be present
  • http.client.duration          → http.method must be present
  • rpc.client.call.duration      → rpc.system.name must be present
  • rpc.client.duration           → rpc.system must be present
            │
     NO ────┴──► Add the missing required attribute to the metric indicated above.
            │
        YES │
            ▼
  Page should be populated. ✓

(3) All hosts showing as "unknown"?
  └──► HTTP v1.23: add server.address to the http.client.request.duration metric.
  └──► HTTP v1.20: add net.peer.name to the http.client.duration metric.
  └──► RPC v1.40: add server.address to the rpc.client.call.duration metric.
  └──► RPC v1.20: add net.peer.name (or server.address as fallback) to the rpc.client.duration metric.

(4) Verify the synthesized metric exists and inspect host values.
```

**Verification queries:**

```sql
-- (1) Check which client metrics are arriving
FROM Metric SELECT count(*) WHERE metricName IN ('http.client.request.duration', 'http.client.duration', 'rpc.client.call.duration', 'rpc.client.duration') AND service.name = 'SERVICE_NAME' FACET metricName SINCE 1 day ago

-- (2) Check required attributes are present on each metric
FROM Metric SELECT count(*) WHERE metricName = 'http.client.request.duration' AND service.name = 'SERVICE_NAME' FACET http.request.method SINCE 1 day ago

-- (3a) Check server.address on http.client.request.duration (missing = "unknown" hosts)
FROM Metric SELECT count(*) WHERE metricName = 'http.client.request.duration' AND service.name = 'SERVICE_NAME' FACET server.address SINCE 1 day ago

-- (3b) Check net.peer.name on http.client.duration (missing = "unknown" hosts for v1.20 HTTP)
FROM Metric SELECT count(*) WHERE metricName = 'http.client.duration' AND service.name = 'SERVICE_NAME' FACET net.peer.name SINCE 1 day ago

-- (3c) Check server.address on rpc.client.call.duration (missing = "unknown" hosts for v1.40 RPC)
FROM Metric SELECT count(*) WHERE metricName = 'rpc.client.call.duration' AND service.name = 'SERVICE_NAME' FACET server.address SINCE 1 day ago

-- (3d) Check net.peer.name on rpc.client.duration (missing = "unknown" hosts for v1.20 RPC; server.address is also tried)
FROM Metric SELECT count(*) WHERE metricName = 'rpc.client.duration' AND service.name = 'SERVICE_NAME' FACET net.peer.name, server.address SINCE 1 day ago

-- (4) Verify synthesized apm.service.external.host.duration exists
FROM Metric SELECT count(*) WHERE metricName = 'apm.service.external.host.duration' AND service.name = 'SERVICE_NAME' FACET external.host SINCE 1 day ago
```

---

### Transaction segment breakdown is blank or inaccurate

```
(1) Are spans being emitted at all?
            │
     NO ────┴──► Enable span export in the OTel SDK configuration.
            │
        YES │
            ▼
(2) Do spans have the correct span.kind?
  • Inbound HTTP/RPC requests → span.kind = server
  • Inbound messaging consumers → span.kind = consumer
  • Outbound DB / HTTP / RPC calls → span.kind = client
            │
     NO ────┴──► span.kind is missing or incorrect on the spans.
            │     Check the instrumentation library — span.kind is usually set automatically
            │     by OTel instrumentation libraries, not manually.
        YES │
            ▼
(3) Are client spans children of the server span in the same trace?
(i.e. DB/HTTP/RPC client spans are nested under the inbound server span)
            │
     NO ────┴──► Context propagation may be broken.
            │     Ensure the OTel SDK has context propagation configured and that
            │     downstream calls inherit the active span context.
        YES │
            ▼
  Breakdown should be visible. ✓

  Numbers look much lower than expected?
  └──► Segment breakdown is extrapolated from sampled spans.
       If only errors are sampled, results will be skewed.
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
  • http.server.request.duration  → add error.type attribute on error requests
  • http.server.duration          → add http.status_code attribute (errors counted when >= 500)
  • rpc.server.call.duration      → add error.type attribute on error requests
  • rpc.server.duration           → add rpc.grpc.status_code attribute (errors counted
                                    when value is in [2, 4, 12, 13, 14, 15])

(2) Verify the synthesized error count metric exists.

Errors Inbox is empty:
(3) Are failed spans setting otel.status_code = ERROR?
  └──► Span exception events alone do NOT mark a span as an error.
       Span status must be explicitly set to ERROR on the span itself.
```

**Verification queries:**

```sql
-- (1a) Check error.type is present on http.server.request.duration
FROM Metric SELECT count(*) WHERE metricName = 'http.server.request.duration' AND service.name = 'SERVICE_NAME' FACET error.type SINCE 1 day ago

-- (1b) Check http.status_code on http.server.duration (errors counted when >= 500)
FROM Metric SELECT count(*) WHERE metricName = 'http.server.duration' AND service.name = 'SERVICE_NAME' FACET http.status_code SINCE 1 day ago

-- (1c) Check error.type on rpc.server.call.duration (v1.40)
FROM Metric SELECT count(*) WHERE metricName = 'rpc.server.call.duration' AND service.name = 'SERVICE_NAME' FACET error.type SINCE 1 day ago

-- (1d) Check rpc.grpc.status_code on rpc.server.duration (errors counted when in [2,4,12,13,14,15])
FROM Metric SELECT count(*) WHERE metricName = 'rpc.server.duration' AND service.name = 'SERVICE_NAME' FACET rpc.grpc.status_code SINCE 1 day ago

-- (2) Verify synthesized apm.service.error.count exists
FROM Metric SELECT count(*) WHERE metricName = 'apm.service.error.count' AND service.name = 'SERVICE_NAME' FACET transactionName SINCE 1 day ago

-- (3) Check error spans are being emitted with otel.status_code = ERROR
FROM Span SELECT count(*) WHERE otel.status_code = 'ERROR' AND service.name = 'SERVICE_NAME' SINCE 1 day ago
```
