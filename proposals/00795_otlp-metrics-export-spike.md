---
issue: https://github.com/praxis-proxy/praxis/issues/795
discussion: https://github.com/praxis-proxy/praxis/issues/795
status: proposed
repos:
  - praxis
authors:
  - henschwartz
graduation_criteria:
  - Maintainer acceptance of spike outcome (`proposed` → `accepted`)
  - Open questions from this proposal answered in the Decisions section
  - Performance posture for periodic OTLP push alongside scrape documented in Architecture recommendation
  - Architecture recommendation documents composite recorder design, config surface, and key integration points sufficient to scope implementation in a follow-up How? PR
  - Optional OTLP metrics export implemented in praxis behind `otel` feature
  - Prometheus `/metrics` scrape remains default and unchanged when OTLP push is off
  - Integration test verifies OTLP metrics reach a collector when enabled
  - Operator docs for shared vs split OTLP endpoints
stakeholders:
  - shaneutt
  - twghu
  - alexsnaps
epic: Observability Epic
related:
  - 00299
  - 00315
  - 00794
  - 00796
  - 00797
experimental_exempt: true
experimental_exempt_reason: >-
  Core metrics recorder and TelemetryConfig wiring in praxis core/protocol
  cannot be prototyped as an external filter in the experimental repo.
---

# OTLP Metrics Export (Alongside Prometheus)

## What?

Praxis today exposes **metrics** exclusively through **Prometheus text
scrape** on the admin `/metrics` endpoint, backed by the `metrics` crate
and `metrics-exporter-prometheus`. **Traces** optionally export over **OTLP**
when the `otel` feature is enabled and `telemetry.otlp_endpoint` (or
`OTEL_EXPORTER_OTLP_ENDPOINT`) is set — see
[#299](https://github.com/praxis-proxy/praxis/issues/299) and
[#315](https://github.com/praxis-proxy/praxis/issues/315).

This proposal, from
[#795](https://github.com/praxis-proxy/praxis/issues/795), is a **spike**
under [Epic #160 Observability](https://github.com/praxis-proxy/praxis/issues/160).
It evaluates **optional OTLP metrics push** so operators who already send
traces to an OpenTelemetry Collector can route **metrics through the same
pipeline** without removing Prometheus scrape. The spike delivers format
evaluation, architecture recommendation, and **Decisions**. **How?** lands in
a follow-up PR after direction is accepted. **Merging this proposal closes
[#795](https://github.com/praxis-proxy/praxis/issues/795)**; code lands in
`praxis` under this proposal when status advances to `accepted`.

### Today's stack (anchor in code)

**Metrics (pull / Prometheus)**

- Global recorder: `install_prometheus_recorder()` in
  `crates/protocol/src/http/pingora/metrics.rs` (also used from filter
  subsystems via `metrics` macros).
- Admin handler renders `PrometheusHandle::render()` on `GET /metrics`.
- Metric families expanded in
  [#794](https://github.com/praxis-proxy/praxis/issues/794) (`praxis_http_*`,
  upstream, config reload, overload, etc.).

**Traces (push / OTLP, optional)**

- `TelemetryConfig` in `crates/core/src/config/telemetry.rs` — endpoint,
  headers, batch interval/size, resource attributes, sampling.
- `init_tracing()` in `crates/core/src/logging.rs` builds
  `SdkTracerProvider` + OTLP gRPC exporter when endpoint resolved.
- **No** `SdkMeterProvider`, **no** `OTEL_EXPORTER_OTLP_METRICS_*` handling,
  **no** bridge from the `metrics` crate to OTel instruments today.

### Spike scope

Evaluate export paths for the **same metric semantics** already emitted
via `metrics` macros:

| Path | Notes |
| --- | --- |
| **Prometheus scrape only (status quo)** | Default; kube-prometheus, OpenShift monitoring, Grafana Agent scrape configs. |
| **OTLP metrics push** | Periodic export to OTel Collector / Grafana Cloud / vendor backends. |
| **Dual export (scrape + push)** | Recommended direction when OTLP is enabled — scrape stays for legacy dashboards. |
| **OTel-native instrumentation only** | Replace `metrics` crate with `opentelemetry::global::meter()` — high migration cost; not v1. |

For each candidate, assess:

- Operator demand in OpenShift / OTel Collector deployments
- Bridge feasibility from existing `metrics` recorder vs rewrite
- Hot-path vs background export cost
- Config sharing with trace OTLP ([#299](https://github.com/praxis-proxy/praxis/issues/299))
- Cardinality and label mapping to OTel semconv

### Spike deliverables

1. **Written comparison** of Prometheus scrape vs OTLP push vs dual export.
2. **Architecture recommendation** — bridge design, config surface, defaults.
3. **Decisions** answering all four questions from
   [#795](https://github.com/praxis-proxy/praxis/issues/795).
4. **How?** — implementation plan for a follow-up PR (not part of the spike
   issue).

### Goals

- Confirm whether OTLP metrics push adds enough value to implement
- Pick a **dual-export** strategy that preserves `/metrics` as default
- Reuse [#299](https://github.com/praxis-proxy/praxis/issues/299) telemetry
  config and OTel resource attributes where possible
- Document performance posture for periodic push alongside scrape
- Document architecture and config surface so implementation can be estimated
  in a follow-up How? PR

### Non-Goals

- **Shipping code in
  [#795](https://github.com/praxis-proxy/praxis/issues/795)** — spike closes
  on this document
- Removing or demoting Prometheus `/metrics` scrape
- Replacing the `metrics` crate instrumentation in v1
- Logs over OTLP (separate from metrics)
- Changing metric names or label sets from [#794](https://github.com/praxis-proxy/praxis/issues/794)
- Auto-configuring Prometheus remote-write

### Open Questions

1. **Demand.** Is there meaningful demand for OTLP metrics **push** over
   Prometheus scrape in Praxis target deployments?
2. **Bridge.** Can the `metrics` crate facade bridge cleanly to an OTel metrics
   reader, or does it require dual-writing / a custom recorder?
3. **Performance.** What is acceptable overhead for periodic OTLP push **alongside**
   existing Prometheus scrape?
4. **Config.** Should metrics OTLP share the same endpoint config as tracing
   OTLP ([#299](https://github.com/praxis-proxy/praxis/issues/299))?

## Why?

### Motivation

Many deployments already run an **OpenTelemetry Collector** for traces
(`telemetry.otlp_endpoint` from [#299](https://github.com/praxis-proxy/praxis/issues/299)).
Those operators still scrape `/metrics` separately — two pipelines, two
label conventions, and duplicate service identity configuration. Unified
OTLP export (traces + metrics) simplifies agent configuration and aligns
with OpenShift cluster monitoring patterns that ingest OTLP on the
collector side.

Prometheus scrape remains valuable: it is the **lowest-friction** path for
local dev, CI, and clusters that already have Prometheus Operator. The
spike therefore targets **optional additive OTLP push**, not a replacement.

The [#299](https://github.com/praxis-proxy/praxis/issues/299) /
[#315](https://github.com/praxis-proxy/praxis/issues/315) prerequisites
named in [#795](https://github.com/praxis-proxy/praxis/issues/795) are
**closed** — trace OTLP plumbing, resource attributes, and batch export
patterns exist and can be mirrored for metrics.

### User Stories

These are stakeholder needs derived from
[#795](https://github.com/praxis-proxy/praxis/issues/795); they are not
separate tracked issues.

- As a platform engineer with an OTel Collector, I want Praxis metrics in
  the same pipeline as traces so that I do not maintain separate scrape and
  push configs for one service.
- As an SRE on Prometheus, I want `/metrics` scrape to keep working unchanged
  when my peer team enables OTLP push.
- As a maintainer, I want a bridge design that does not rewrite every
  `counter!` / `histogram!` callsite from [#794](https://github.com/praxis-proxy/praxis/issues/794).
- As a release owner, I want OTLP metrics **off by default** so default
  installs behave exactly as today.

## Export path comparison

This section satisfies spike deliverable (1).

| Path | Strengths | Weaknesses | Praxis fit |
| --- | --- | --- | --- |
| **Prometheus scrape only** | Universal in K8s; no push auth complexity; zero background export tasks; works offline | Requires scrape config per pod; harder through strict egress; separate from trace pipeline | **Today’s default** — keep |
| **OTLP push only** | Unified collector; works when scrape is blocked; same resource attrs as traces | Breaks Prometheus-only operators; needs push interval tuning; another failure mode | **Reject as default** |
| **Dual export (scrape + OTLP push)** | Best of both; OTLP optional; scrape unchanged | Two export paths to test; bridge complexity | **Recommended v1** |
| **Migrate to OTel `Meter` API** | Native OTLP instruments; no bridge | Rewrites all metric callsites; duplicates Prometheus exposition unless OTel Prometheus exporter added | **Defer** — too costly for v1 |

### Summary recommendation

**Default:** Prometheus `/metrics` scrape unchanged.

**Optional:** When `telemetry.otlp_metrics_enabled: true` and a metrics OTLP
endpoint is resolved (shared trace endpoint or
`OTEL_EXPORTER_OTLP_METRICS_ENDPOINT`), install an **`SdkMeterProvider`**
with **`PeriodicReader`** + OTLP exporter, fed by a **composite / fan-out
recorder** behind the existing `metrics` crate macros.

**Do not** remove `metrics-exporter-prometheus` when OTLP is on.

## Architecture recommendation

This section satisfies spike deliverable (2).

### Recommended v1 design

```
Request path                    Background
───────────                    ───────────
counter!/histogram!  ──►  metrics::Recorder (composite)
                              ├─► Prometheus recorder (existing)
                              └─► OTel metric sync (new, if enabled)
                                        │
                                        ▼
                              PeriodicReader (e.g. 60s)
                                        │
                                        ▼
                              OTLP/gRPC metrics exporter
                                        │
                                        ▼
                              OTel Collector (same as traces)
```

**Recorder strategy (bridge):** The `metrics` crate allows **one** global
recorder. v1 should install a **composite recorder** at startup that:

1. Delegates to the existing `PrometheusBuilder` recorder (unchanged render
   path for `/metrics`).
2. When OTLP metrics enabled, maps `metrics` `Counter` / `Gauge` /
   `Histogram` updates to OTel instruments with equivalent names and labels.
   **Histogram parity:** Praxis configures non-default Prometheus histogram
   buckets (for example `BODY_SIZE_BUCKETS_BYTES` in
   `crates/protocol/src/http/pingora/metrics.rs`). The composite recorder must
   register matching OTel histogram **views** / explicit boundaries on
   `SdkMeterProvider` instruments — not OTel SDK defaults — so OTLP-exported
   histograms match `/metrics` bucket boundaries on dual export.

This is **dual-write at record time**, not a post-hoc scrape conversion.
It avoids rewriting every callsite and avoids replacing Prometheus exposition.

**Alternative rejected for v1:** Read Prometheus text from `/metrics` and
forward to collector — stale, scrape-interval dependent, and duplicates work.

**Alternative rejected for v1:** OTel Prometheus exporter as the only sink —
changes operator contract and loses current histogram bucket customization
(`BODY_SIZE_BUCKETS_BYTES`, etc.).

### Config surface (proposed)

Extend `TelemetryConfig` (same module as traces):

| Field | Default | Purpose |
| --- | --- | --- |
| `otlp_metrics_enabled` | `false` | Opt-in OTLP metrics push |
| `otlp_metrics_interval_secs` | `60` | `PeriodicReader` interval; **`u64` seconds**; validation: **`≥ 10`** and **`≤ 3600`** (reject at config load; sub-10s values are inadvisable for production — risk collector overload) |
| `otlp_endpoint` | (existing) | **Shared** trace + metrics endpoint when metrics-specific env unset |
| `otlp_headers` | (existing) | Shared auth headers |
| `service_name`, `service_version`, `environment` | (existing) | Shared resource attributes |

**Environment precedence (OTel spec aligned):**

- Metrics endpoint: `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` → else
  `telemetry.otlp_endpoint` / `OTEL_EXPORTER_OTLP_ENDPOINT`
- Metrics protocol: `OTEL_EXPORTER_OTLP_METRICS_PROTOCOL` → else shared
  `OTEL_EXPORTER_OTLP_PROTOCOL` (default gRPC)

Traces and metrics **share** resource attributes and headers; **may differ**
on endpoint only when operators set metrics-specific env vars (common for
single collector with different ports is rare — same host:4317 is typical).

### Cardinality / naming

- Keep **`praxis_*` metric names** and existing label keys (`method`,
  `status_class`, `cluster`, …) in both exporters for parity.
- **Histogram bucket boundaries** must match between Prometheus and OTLP paths
  (same boundaries as today's `PrometheusBuilder` / `BODY_SIZE_BUCKETS_*`
  configuration). The follow-up How? PR must wire OTel histogram instruments
  with those boundaries; silent OTel-default buckets are not acceptable for v1.
- OTel semconv mapping is a **collector concern** (transform processor), not
  a Praxis v1 requirement.

### Performance posture (qualitative)

| Concern | Assessment |
| --- | --- |
| Request hot path | Composite recorder adds one extra atomic/map update per emission — same order as today’s Prometheus recorder; acceptable. |
| Background push | Default **60s** interval; bounded work proportional to active metric cardinality (already controlled). |
| Memory | Second aggregation layer in OTel SDK — monitor in implementation; cardinality rules from [#794](https://github.com/praxis-proxy/praxis/issues/794) apply. |
| Export failures | When the collector is unreachable, `PeriodicReader` retries on a background task. **Export failures must not affect request processing** or Prometheus scrape. The follow-up How? PR must bound OTel-side aggregation memory (SDK/exporter defaults or explicit caps) so sustained export outage cannot grow unbounded in-process state; on repeated failure, drop or reset OTel aggregates rather than blocking the proxy or disabling `/metrics`. |
| Scrape | Unchanged — no extra work on `/metrics` requests beyond today. |

Micro-benchmarks belong in the implementation PR if stakeholders require
numbers; not spike blockers.

## Decisions

Answers to **Open Questions** above. Spike outcomes; a follow-up **How?** PR
covers implementation after this proposal is `accepted`.

1. **Demand.** **Yes, for a subset of deployments** — teams already exporting
   traces via OTLP ([#299](https://github.com/praxis-proxy/praxis/issues/299))
   consistently ask for metrics on the same collector path. Prometheus scrape
   remains the majority default in bare-metal and Prometheus Operator shops.
   **Implement optional OTLP push**; do not prioritize push-only deployments.

2. **Bridge.** **Custom composite recorder** dual-writing to
   `metrics-exporter-prometheus` and OTel `SdkMeterProvider` instruments,
   including **matching histogram bucket boundaries** on the OTel side (see
   **Architecture recommendation** and **Cardinality / naming**). The
   `metrics` crate does **not** offer a built-in OTel reader. Full migration to
   OTel `Meter` API is **deferred**.

3. **Performance.** **Acceptable** with periodic push (default 60s) alongside
   scrape. Hot-path cost is one composite record op; push work is off-thread.
   No request-path HTTP to collector. **Export failures are off-thread** and
   must not block proxying; OTel aggregation memory must stay bounded when the
   collector is down (see **Performance posture**).

4. **Config.** **Share** endpoint, headers, and resource attributes with trace
   OTLP by default ([#299](https://github.com/praxis-proxy/praxis/issues/299)).
   Honor **`OTEL_EXPORTER_OTLP_METRICS_*`** overrides per OTel specification.
   Add explicit `telemetry.otlp_metrics_enabled` so metrics push is **opt-in**
   even when trace export is on.

## Spike completion

Merging this proposal **closes
[#795](https://github.com/praxis-proxy/praxis/issues/795)**. All issue
questions are answered in **Decisions**; comparison and architecture are in
**Export path comparison** and **Architecture recommendation**.

| #795 question | Section |
| --- | --- |
| Demand for OTLP push vs scrape | **Decisions** #1, **Export path comparison** |
| `metrics` crate bridge vs dual-write | **Decisions** #2, **Architecture recommendation** |
| Performance overhead | **Decisions** #3, **Performance posture** |
| Shared endpoint with trace OTLP | **Decisions** #4, **Config surface** |

Implementation is tracked under this proposal's graduation criteria when status
advances to `accepted`, not under #795.
