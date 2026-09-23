---
issue: https://github.com/praxis-proxy/praxis/issues/784
discussion: https://github.com/praxis-proxy/praxis/issues/784
status: proposed
repos:
  - praxis
authors:
  - henschwartz
graduation_criteria:
  - Maintainer acceptance of spike outcome (`proposed` → `accepted`)
  - Dedicated `audit_log` filter and Praxis Audit Record v1 emission in
    praxis (implementation PR under this proposal)
  - Unit and integration tests plus example config
  - Filter reference documentation
stakeholders:
  - shaneutt
  - twghu
  - alexsnaps
epic: Observability Epic
related:
  - 00126
  - 00317
  - 00799
  - 00797
  - 00798
  - 00796
  - 00794
origin:
  repo: praxis
  issue: https://github.com/praxis-proxy/praxis/issues/784
  file: 00784_structured-security-audit-log-format.md
  pr: https://github.com/praxis-proxy/praxis/pull/1004
---

# Structured Security Audit Log Format Evaluation

## What?

Praxis today surfaces security-relevant decisions through a mix of
**per-request access logs**, **response headers**, and **process
logs**. That is enough for debugging but not for compliance-oriented
deployments that need a **dedicated, machine-parseable audit trail**
of allow/deny decisions with stable identifiers and SIEM-ready schemas.

This proposal, from
[#784](https://github.com/praxis-proxy/praxis/issues/784), is a
**spike** under
[Epic #160 Observability](https://github.com/praxis-proxy/praxis/issues/160).
It evaluates standardized security event formats and recommends how
Praxis should emit **structured security audit records** for
request-path denials and policy decisions. The spike delivers format
evaluation, schema draft, sink architecture, Decisions, and an
implementation design (**How?**). **Merging this proposal closes
[#784](https://github.com/praxis-proxy/praxis/issues/784)**; code
lands in `praxis` under this proposal when status advances to
`accepted`.

### Today’s signals (anchor in code)

**Request-path denials and policy outcomes**

- **`policy` filter** — rejects with stable violation codes on the
  `X-Policy-Violation` response header (for example
  `auth.invalid_token`, `apl.policy`, `pii.detected`). CPEX policy
  violations use the `apl.policy` namespace; `policy.deny` is the
  gateway-local fallback in `http_authz_rejection` when no specific
  violation is supplied, not a policy-engine code.
- **`basic_auth`** — HTTP 401 with `WWW-Authenticate`
- **`ip_acl`** — allow/deny by client IP
- **`rate_limit`** — HTTP 429 when quotas are exceeded
- **`csrf`** — reject or log-only mode for cross-origin violations
- **`guardrails`** — configurable reject on policy match
- **`cors`** — reject mode returns HTTP 403 on origin violations
- **`peer_identity_trust`** — HTTP 403 when mTLS peer identity does not
  match trusted peers

**Out of v1 audit scope (issue summary vs code)**

- **`credential_injection`** — fetches and injects outbound credentials
  (`FilterAction::Continue` only); it does **not** reject requests.
  [#784](https://github.com/praxis-proxy/praxis/issues/784) mentions
  credential injection in its summary; v1 audit records **traffic
  denials and policy rejections**, not outbound credential lifecycle
  events. Those belong in a separate control-plane or outbound-audit
  track if needed later.

**Observability overlap (not a dedicated audit trail)**

- **`access_log` filter** — structured JSON per request via
  `tracing::info!`; volume-oriented, not scoped to security events.
  Field selection and emit-time conditions are proposed in
  [#799](https://github.com/praxis-proxy/praxis/issues/799); that
  improves access visibility but does not replace an audit schema.
- **Process logging** — for example `config reload audit` in
  `server/src/reload_diagnostics.rs` (listener/cluster/filter-chain
  diffs). That is **control-plane** audit, not per-request security
  enforcement.

**Downstream context**

Agent runtimes (for example OpenShell) audit every allow/deny
decision. When Praxis serves as an egress gateway, enterprise SIEM
pipelines expect **standardized event schemas**, not ad hoc access-log
fields or grep of `X-Policy-Violation` in response captures.

### Spike scope

Evaluate these format families for **denial / policy-decision events**:

| Format | Notes |
| --- | --- |
| **OCSF** | Open Cybersecurity Schema Framework — e.g. class 6002 (API Activity), class 4002 (HTTP Activity). Versioned, vendor-backed (AWS, Splunk, IBM). |
| **ECS** | Elastic Common Schema — common in Elastic/OpenSearch stacks. |
| **CloudEvents** | CNCF envelope for event metadata; payload schema-agnostic. |
| **Custom structured JSON** | Extend Praxis conventions (violation code, filter name, identity hints, request metadata) without an external schema registry. |

For each candidate, the spike should assess:

- Fit for **denial-focused** records (not full access-log volume)
- Field mapping from Praxis sources above (especially `policy` /
  `X-Policy-Violation`)
- SIEM/OpenShift consumption path for target deployments
- Serialization cost on the hot path (qualitative; micro-benchmark only
  if needed to break a tie)
- Whether output belongs in **access logs**, a **dedicated audit
  sink**, or both

### Spike deliverables (acceptance)

1. **Written comparison** of the formats with pros/cons for Praxis
   (OpenShift egress, SIEM forwarding, operator skill sets).
2. **Schema draft** for the recommended format, including at least one
   **sample denial event** (for example `apl.policy` or
   `auth.invalid_token`; gateway fallback `policy.deny` when no code
   is supplied) with realistic field values.
3. **Sink architecture recommendation** — separate file/syslog/Kafka
   sink vs extension of access log vs process log — with rationale and
   interaction with
   [#797](https://github.com/praxis-proxy/praxis/issues/797) (process
   log destinations) and
   [#126](https://github.com/praxis-proxy/praxis/issues/126) (custom
   access-log formats / multi-sink).

### Goals

- Pick a **default audit schema direction** (or a short prioritized
  list) that Praxis can implement without re-litigating formats per
  filter
- Define a **minimum denial record** operators and SIEM teams can rely
  on (identity, outcome, violation/rule id, request metadata,
  timestamp, correlation ids)
- Clarify **audit log vs access log** boundaries so
  [#799](https://github.com/praxis-proxy/praxis/issues/799) and
  [#126](https://github.com/praxis-proxy/praxis/issues/126) do not
  absorb compliance audit requirements by accident
- Document enough implementation detail (**How?**) that engineering
  can estimate and ship without re-opening format or sink questions

### Non-Goals

- **Shipping code in
  [#784](https://github.com/praxis-proxy/praxis/issues/784)** — the
  issue is a spike; closure is this proposal, not a `praxis` PR
- Replacing or redesigning the `access_log` filter
- Full
  [#126](https://github.com/praxis-proxy/praxis/issues/126) scope
  (templates, per-route formats, syslog/multi-sink productization) —
  only whether [#126](https://github.com/praxis-proxy/praxis/issues/126)
  could **carry** audit events
- **Security audit log for control-plane** events (config reload,
  admin API mutations) — note interactions but out of v1 request-path
  scope unless the spike finds a unified envelope is trivial
- TCP audit parity (HTTP first; align with
  [#799](https://github.com/praxis-proxy/praxis/issues/799) /
  access-log theme)
- Defining retention, encryption, or tamper-evidence for audit files
- Evaluating every OCSF category — focus on API/HTTP activity and
  authentication/authorization outcomes relevant to proxy denials

### Open Questions

1. **Consumer formats.** Which format(s) do target deployment
   environments (OpenShift, enterprise SIEM, SOC tooling) actually
   ingest today — OCSF, ECS, CloudEvents wrapper, or custom JSON?
2. **Minimum fields.** What fields are required for a useful denial
   audit record? (identity/subject, policy rule, violation code,
   filter name, method, path, client IP, status, `request_id`,
   `trace_id`, timestamp, outcome)
3. **Sink model.** Should audit events use a **separate output sink**
   (dedicated file, syslog, Kafka) or extend the existing access log
   / process log paths?
4. **Hot-path cost.** What is acceptable serialization overhead for
   denial-only emission (expected low volume vs access log)?
5. **Vehicle vs dedicated filter.** Can
   [#126](https://github.com/praxis-proxy/praxis/issues/126) serve as
   the implementation vehicle, or does audit logging need a
   **dedicated filter** (for example `audit_log`) with its own
   conditions and schema guarantees?

## Why?

### Motivation

Access logs answer “what traffic flowed through the proxy?” Security
audit answers “**which policy decisions were made, by whom, on what
resource, with what outcome?**” Those questions overlap on denied
requests but diverge on allowed traffic volume, retention, and schema
rigor.

Today, operators stitch together:

- `access_log` lines (if enabled and sampled)
- `X-Policy-Violation` on individual responses
- Ad hoc process-log messages

That breaks down for compliance and centralized SIEM: fields are
inconsistent across filters, allow traffic drowns denials in access
logs, and there is no stable **event type** or **outcome** taxonomy
across `policy`, `ip_acl`, `rate_limit`, and siblings.

A structured audit format positions Praxis as a credible **egress
gateway** for agent workloads and regulated environments, and it
complements (rather than duplicates) work already in flight on process
logging ([#797](https://github.com/praxis-proxy/praxis/issues/797)),
runtime log levels
([#798](https://github.com/praxis-proxy/praxis/issues/798)), and
access-log field selection
([#799](https://github.com/praxis-proxy/praxis/issues/799)).

Trace correlation ([#317](https://github.com/praxis-proxy/praxis/issues/317))
should appear in the recommended schema where OTel is active, but this
spike does not implement correlation — it specifies how audit records
**relate** to traces and access logs.

### User Stories

These are stakeholder needs derived from
[#784](https://github.com/praxis-proxy/praxis/issues/784); they are
not separate tracked issues.

- As a security operator, I want denial events in a **standard schema**
  so that my SIEM can alert on policy violations without custom
  parsers per filter.
- As a compliance reviewer, I want a **dedicated audit trail** of
  enforcements (not sampled access traffic) so that reviews can
  prove what was blocked and why.
- As a platform engineer running Praxis on OpenShift, I want audit
  output that fits **existing log forwarders** (stdout/file/syslog →
  cluster logging stack) without forking Praxis.
- As an SRE, I want audit emission to be **low overhead** because
  only denials and explicit policy events are recorded, not every
  `200 OK`.
- As a Praxis maintainer, I want a spike outcome that clearly states
  whether audit belongs in
  [#126](https://github.com/praxis-proxy/praxis/issues/126), a new
  filter, or the process-log
  pipeline so we do not build three incompatible paths.

## Format comparison

This section satisfies spike deliverable (1) and maps to the format
families listed under **Spike scope** above.

| Format | Strengths for Praxis | Weaknesses for Praxis | OpenShift / SIEM fit |
| --- | --- | --- | --- |
| **OCSF** | Versioned schema; strong vendor support (AWS Security Lake, Splunk, IBM QRadar); HTTP/API activity classes (4002, 6002) cover proxy denials | Heavy mapping from Praxis-specific signals (`X-Policy-Violation`, JSON-RPC policy envelopes, MCP vs generic HTTP deny paths); schema churn across OCSF versions; over-specified for a v1 spike | Good when the downstream pipeline is already OCSF-native; requires a transform layer from Praxis-native records |
| **ECS** | Ubiquitous in Elastic/OpenSearch stacks; `event.*`, `http.*`, `source.*`, `user.*` fields map cleanly to denial metadata | Less opinionated about authorization outcomes; no first-class `violation_code` for policy engines; Elastic-centric | Excellent for clusters shipping logs to ECK / OpenSearch; same transform-layer story as OCSF |
| **CloudEvents** | CNCF-standard envelope (`id`, `source`, `type`, `time`, `datacontenttype`); transport-agnostic; Kafka-friendly | Does **not** define security payload semantics — Praxis would still need an inner schema | Useful as an **optional wrapper** around a Praxis payload for Kafka/event buses; not sufficient alone |
| **Custom structured JSON (Praxis Audit Record v1)** | Maps directly from existing Praxis signals (`X-Policy-Violation`, filter name, `request_id`, access-log field conventions); stable `audit_version`; denial-only volume | Not SIEM-native without a documented mapping to OCSF/ECS; operators must configure forwarder rules on `message=audit` or JSON field | **Best default for v1**: stdout/file via cluster log forwarder (OpenShift Logging / Loki / Fluent Bit) with optional downstream normalization |

### Summary recommendation

**Primary native format:** **Praxis Audit Record v1** (custom structured JSON
with a versioned `audit_version` field).

**Secondary interoperability:** document **field mapping tables** from Audit
Record v1 → OCSF class 4002 (HTTP Activity) and ECS `event`/`http`/`rule`
fields so SIEM teams can normalize in the collector. Do **not** emit OCSF/ECS
natively in v1 — the mapping cost and Praxis-specific violation taxonomy
(`apl.policy`, `auth.invalid_token`, filter-scoped fallbacks) fit a
Praxis-owned schema better.

**Optional envelope:** CloudEvents wrapping is a
[#126](https://github.com/praxis-proxy/praxis/issues/126) / multi-sink
concern, not a spike blocker.

## Schema draft

This section satisfies spike deliverable (2). The recommended format is
**Praxis Audit Record v1** — a single JSON object per denial event.

### Required fields (v1 minimum denial record)

Top-level fields plus one required nested `http` object (JSON shape matches
the samples below — table lists nested keys as `http.<field>` for readability).

| Field | Type | Source in Praxis |
| --- | --- | --- |
| `audit_version` | string | Constant `"1"` |
| `event_type` | string | Constant `"security.denial"` |
| `timestamp` | string (RFC 3339 UTC) | Emit time |
| `outcome` | string | Constant `"deny"` for request-path rejections |
| `filter` | string | Denying filter name (`policy`, `ip_acl`, …) |
| `violation_code` | string | `X-Policy-Violation` when present; otherwise filter-scoped default (see below) |
| `http` | object | Required nested object; subfields below |
| `http.method` | string | Request method (nested key `method` under `http`) |
| `http.path` | string | Sanitized request path (nested key `path` under `http`; see **Path sanitization (v1)**) |
| `http.status` | number | HTTP status on the rejection response (nested key `status` under `http`) |
| `client_ip` | string | Client address |
| `listener` | string | Listener name |
| `request_id` | string | Praxis request id |

**Path sanitization (v1).** `http.path` is the request path **without the
query string** (`?…` stripped). Use the path as seen by Praxis after Pingora
normalization (no extra `../` resolution beyond what the proxy already
applies). v1 does **not** redact UUID/email-like segments in path components.

### Optional fields (emit when available)

| Field | When |
| --- | --- |
| `trace_id`, `span_id` | OpenTelemetry active ([#317](https://github.com/praxis-proxy/praxis/issues/317)) |
| `cluster` | Upstream cluster selected before denial |
| `peer_identity` | mTLS / `peer_identity_trust` denials |

### Violation code resolution

1. Prefer **`X-Policy-Violation`** response header when the denying filter
   sets it (especially `policy`).
2. Otherwise use **filter-scoped defaults**:

| Filter | Default `violation_code` | Typical `http.status` |
| --- | --- | --- |
| `policy` | `policy.deny` (gateway fallback when header absent) | 401 / 403 / 200 (JSON-RPC envelope) |
| `basic_auth` | `basic_auth.required` | 401 |
| `ip_acl` | `ip_acl.denied` | 403 |
| `rate_limit` | `rate_limit.exceeded` | 429 |
| `csrf` | `csrf.violation` | 403 |
| `cors` | `cors.origin_denied` | 403 |
| `guardrails` | `guardrails.denied` | 403 |
| `peer_identity_trust` | `peer_identity.untrusted` | 403 |

### Sample events

**Policy denial** (`auth.invalid_token` from `policy` filter):

```json
{
  "audit_version": "1",
  "event_type": "security.denial",
  "timestamp": "2026-09-06T12:34:56.789Z",
  "outcome": "deny",
  "filter": "policy",
  "violation_code": "auth.invalid_token",
  "http": {
    "method": "POST",
    "path": "/v1/messages",
    "status": 401
  },
  "client_ip": "10.0.0.42",
  "listener": "egress",
  "request_id": "req-7f3a2b1c",
  "trace_id": "4bf92f3577b34da6a3ce929d0e0e4736",
  "span_id": "00f067aa0ba902b7",
  "cluster": "anthropic-upstream"
}
```

**CPEX policy denial** (`apl.policy` from `policy` filter; MCP clients
may receive HTTP 200 with a JSON-RPC error envelope and
`X-Policy-Violation: apl.policy`):

```json
{
  "audit_version": "1",
  "event_type": "security.denial",
  "timestamp": "2026-09-06T12:34:58.100Z",
  "outcome": "deny",
  "filter": "policy",
  "violation_code": "apl.policy",
  "http": {
    "method": "POST",
    "path": "/mcp/v1/sessions/abc/tools/call",
    "status": 200
  },
  "client_ip": "10.0.0.42",
  "listener": "egress",
  "request_id": "req-2a8b4c9d"
}
```

**IP ACL denial** (no `X-Policy-Violation`; filter default code):

```json
{
  "audit_version": "1",
  "event_type": "security.denial",
  "timestamp": "2026-09-06T12:35:01.002Z",
  "outcome": "deny",
  "filter": "ip_acl",
  "violation_code": "ip_acl.denied",
  "http": {
    "method": "GET",
    "path": "/health",
    "status": 403
  },
  "client_ip": "203.0.113.50",
  "listener": "web",
  "request_id": "req-9c8d7e6f"
}
```

### Mapping notes (downstream normalization)

For teams requiring OCSF or ECS, the recommended collector transform is:

- OCSF: class **4002** (HTTP Activity), `activity_id` = deny,
  `status_code` = `http.status`, `url` = path, `actor` from
  `peer_identity` or client IP, `unmapped` carries `violation_code`
- ECS: `event.category` = `network`, `event.type` = `denied`,
  `event.outcome` = `failure`, `rule.name` = `violation_code`,
  `http.request.method`, `url.path`, `source.ip`

**Detection rule note.** Downstream SIEM rules must key on
`outcome == "deny"` or `event_type == "security.denial"`, **not**
`http.status >= 400`. CPEX / JSON-RPC policy denials (see sample above)
can return HTTP **200** with `outcome: "deny"` and
`violation_code: "apl.policy"`.

## Sink architecture

This section satisfies spike deliverable (3).

### Recommendation (v1)

Use a **dedicated audit emission path** that is **logically separate**
from `access_log`, but **physically colocated** with today's tracing
subscriber (stdout / process log):

1. **Dedicated `audit_log` filter (marker)** in the HTTP pipeline — when
   present, security filter denials emit an audit record. When absent,
   denials behave as today (no audit JSON).
2. **Emit via `tracing::info!`** with a stable discriminator
   (`message = "audit"`, structured `record` field holding the JSON
   payload) so operators can route audit lines separately from access
   lines (`message = "access"` or equivalent) in Fluent Bit / Vector /
   OpenShift Logging pipelines.
3. **Process log destinations**
   ([#797](https://github.com/praxis-proxy/praxis/issues/797)) apply to
   the same subscriber — audit and access share the process output
   target but remain **distinct event streams** via message / JSON
   field filters.
4. **Do not extend `access_log`** for compliance audit — access logs
   are volume-oriented, optionally sampled
   ([#799](https://github.com/praxis-proxy/praxis/issues/799)), and
   lack a stable security event taxonomy. Mixing audit into access
   logs would force operators to sample or filter compliance events
   accidentally.
5. **Do not rely on `#126` as the v1 vehicle** — customizable formats
   and multi-sink productization belong in
   [#126](https://github.com/praxis-proxy/praxis/issues/126); audit
   needs a **schema guarantee** and denial-only trigger independent of
   access-log templates.

### OpenShift deployment path

```
Praxis pod stdout
  → cluster log collector (Fluent Bit / Vector)
    → route where message == "audit" OR json.event_type == "security.denial"
      → SIEM / long-retention store
    → route access logs separately (existing dashboards)
```

Dedicated file/syslog/Kafka sinks are **out of v1 implementation scope**
here but align with future
[#126](https://github.com/praxis-proxy/praxis/issues/126) work — the
v1 design must not block adding sinks later.

### Performance posture (qualitative)

Denial volume is orders of magnitude below access-log volume. One
`serde_json` serialization per denial on the reject path is acceptable
for v1. Micro-benchmarks belong in the implementation PR if
stakeholders require numeric proof; they are not spike blockers.

## Decisions

Answers to the **Open Questions** above. These lock spike outcomes;
the **How?** section below specifies implementation when this
proposal is `accepted`.

1. **Consumer formats.** **Praxis Audit Record v1 (custom JSON)** as
   the native format. Document OCSF/ECS mapping for downstream
   collectors. CloudEvents wrapping deferred to
   [#126](https://github.com/praxis-proxy/praxis/issues/126) /
   multi-sink work.
2. **Minimum fields.** Required set in **Schema draft** above
   (`audit_version`, `event_type`, `timestamp`, `outcome`, `filter`,
   `violation_code`, `http.*`, `client_ip`, `listener`,
   `request_id`). Optional: `trace_id`, `span_id`, `cluster`,
   `peer_identity`. Identity/subject: v1 uses `client_ip` and
   `peer_identity` when present; richer subject claims from JWT/policy
   context are a future extension.
3. **Sink model.** **Separate logical stream** via dedicated
   `audit_log` filter + tracing discriminator; **not** an extension of
   `access_log`. Physical output uses the existing process log
   pipeline ([#797](https://github.com/praxis-proxy/praxis/issues/797));
   dedicated file/syslog/Kafka sinks are follow-on via
   [#126](https://github.com/praxis-proxy/praxis/issues/126).
4. **Hot-path cost.** **Denial-only emission** with single JSON
   serialize per event is acceptable for v1. No access-log-scale
   sampling on audit records.
5. **Vehicle vs dedicated filter.** **`audit_log` dedicated marker
   filter** with pipeline-gated emission and stable schema — **not**
   [#126](https://github.com/praxis-proxy/praxis/issues/126) and **not**
   `access_log` extension.

## How?

Implementation ships in `praxis-proxy/praxis` under this proposal
after maintainers accept the spike outcome (`proposed` → `accepted`).
This section is **design only** — it does not block closing
[#784](https://github.com/praxis-proxy/praxis/issues/784).

### Implementation

One implementation PR series in `praxis` covering:

- `audit_log` marker filter (empty v1 config — presence gates emission)
- Pipeline hook on security-filter reject paths
- `maybe_emit_security_audit` helper building Praxis Audit Record v1
- `tracing::info!(message = "audit", record = %json)` emit path
- `listener_name` on `HttpFilterContext` (from Pingora handler)
- Unit tests for violation-code resolution and record shape
- Example config + integration test (denial returns 403 **and** captured
  `message=audit` JSON matches v1 record)
- Filter reference / observability docs

**Key files**

- `crates/filter/src/builtins/http/observability/audit_log.rs` — filter,
  record builder, violation-code resolution, emit helper
- `crates/filter/src/builtins/http/observability/mod.rs` — exports
- `crates/filter/src/registry.rs` — register `audit_log`
- `crates/filter/src/pipeline/http.rs` — call emit helper on reject
  paths (request, response, body, branch)
- `crates/filter/src/context.rs` — `listener_name: Option<Arc<str>>`
- `crates/protocol/src/http/pingora/context.rs` — `listener_name` on
  `PingoraRequestCtx`
- `crates/protocol/src/http/pingora/handler/with_body.rs` — populate
  `listener_name` at request start
- `examples/configs/observability/audit-log.yaml` — example
- `tests/integration/tests/suite/audit_log.rs` — functional test
- `docs/filters/reference.md` (or observability section) — operator
  docs

### Design

**Anchor in today's stack.** Security filters return
`FilterAction::Reject` from `on_request` (and occasionally later
phases). The HTTP pipeline executor in `pipeline/http.rs` already
centralizes reject handling. `access_log` in
`observability/access_log.rs` shows the `tracing::info!` structured
emit pattern. Policy violations already stamp
`X-Policy-Violation` (`policy/error.rs`).

**`audit_log` filter (marker).** v1 config is empty
(`filter: audit_log` only). The filter's `on_request` returns
`Continue` — it does not emit per-request. Instead, the pipeline
checks whether `audit_log` is configured in the active pipeline
before calling the emit helper on security denials. **Filter chain
position does not matter** — `audit_log` is a pipeline-level marker;
denials from any security filter in the same pipeline are captured
whether `audit_log` appears before or after that filter in YAML.

**Security filters in scope (v1).** Emit audit records when these
filters reject and `audit_log` is present:

- `ip_acl`, `rate_limit`, `csrf`, `cors`, `guardrails`,
  `peer_identity_trust`
- `policy` and `basic_auth` when compiled with their feature flags

**Emit trigger.** On any pipeline reject where the denying filter is
in the security set above, call `maybe_emit_security_audit(ctx,
filter_name, rejection)`. Skip when `audit_log` is absent from the
pipeline (today's behavior).

**Violation code resolution** (matches **Schema draft**):

1. Read `X-Policy-Violation` from the rejection response headers when
   set.
2. Else use filter-scoped defaults (`ip_acl.denied`,
   `rate_limit.exceeded`, …).

**Record build.** Serialize **Praxis Audit Record v1** (see **Schema
draft**) with RFC 3339 UTC `timestamp`, path per **Path sanitization
(v1)**, client IP, listener name, `request_id`, optional OTel ids from
active span, and optional `cluster` / `peer_identity` from context.

**Tracing emit.**

```rust
info!(message = "audit", record = %json);
```

Operators route on `message == "audit"` or `event_type ==
"security.denial"` (see **Sink architecture**).

**Config example** (illustrative ordering only — `audit_log` position
in the chain does not gate which filters emit audit records).

```yaml
filter_chains:
  - name: main
    filters:
      - filter: ip_acl
        allow: ["10.0.0.0/8"]
      - filter: audit_log
      - filter: router
        routes:
          - path_prefix: "/"
            cluster: backend
```

**Tests.**

- Unit: violation-code resolution (header vs defaults); record JSON
  shape; skip emit when `audit_log` not in pipeline
- Integration: `ip_acl` deny + `audit_log` in pipeline → HTTP 403 **and**
  **required** verification that an audit record was emitted: capture process
  log / tracing output in-test and assert **exactly one** line with
  `message=audit` whose `record` JSON matches Praxis Audit Record v1 for
  that denial (`event_type`, `outcome: deny`, `violation_code`, `http.status`,
  `client_ip`). HTTP 403 alone is **insufficient** — regressions that stop
  emitting audit JSON while still denying must fail the test.
- Schema: example config parses

**Explicitly out of this How (see Non-Goals):** dedicated
file/syslog/Kafka sinks
([#126](https://github.com/praxis-proxy/praxis/issues/126)), TCP audit
parity, control-plane audit events, OCSF/ECS native emission, allow-path
audit (only denials in v1).

## Spike completion

Merging this proposal **closes
[#784](https://github.com/praxis-proxy/praxis/issues/784)**. All issue
acceptance criteria are satisfied in this document:

| #784 criterion | Section |
| --- | --- |
| Written format comparison | **Format comparison** |
| Schema draft + sample denial event | **Schema draft** |
| Sink architecture recommendation | **Sink architecture** |

Open questions are answered in **Decisions**. Implementation design is
in **How?** — tracked under this proposal's graduation criteria when
status advances to `accepted`, not under #784.
