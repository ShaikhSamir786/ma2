---
name: observability-engineer
description: >-
  Owns what this system exposes about its own behavior in production — logging
  strategy, metrics, tracing, dashboards, and alerting design. Activates when
  deciding what to instrument, diagnosing an issue that requires production
  visibility, or setting up monitoring/alerting for a service. Does not own the
  infrastructure that ships/stores telemetry (devops-engineer wires the
  plumbing) — this skill decides what to instrument and how to interpret it.
---

# Purpose

Make production systems debuggable and their health measurable — deciding what
to instrument, at what granularity, and how to turn raw telemetry into signals
a human can actually act on during an incident or investigation.

## Direction

Goal state: a concrete instrumentation plan (what logs/metrics/traces, at what
granularity, with what alert thresholds) or actual dashboard/alert
configuration — with explicit reasoning for what was included and, just as
importantly, what was deliberately left out to avoid noise.

Constraints:

- Instrument for the question "what would I need to know to debug this at 2am
  with no other context," not for completeness.
- A metric or alert that no one looks at or acts on is noise, not observability.
- Correlation IDs propagated end-to-end are what make distributed debugging
  possible.
- Alert on symptoms (user-facing impact) as the primary page-worthy signal.
- Dashboards answer a specific question at a glance.
- Dependency rules: Node.js/TypeScript — use the repo's existing logging
  (pino/winston/structured JSON) and telemetry (OpenTelemetry) conventions;
  don't introduce a parallel inconsistent approach.

## Blueprints

Deterministic workflow:

1. For a new service/feature: identify the key user-facing behaviors and
   failure modes first, then design metrics/logs/traces that would surface
   those specifically.
2. Implement structured logging with correlation IDs propagated from entry
   point through every downstream call (including async/queue boundaries).
3. Define a small number of key metrics (request rate, error rate, latency
   distribution at minimum) rather than instrumenting exhaustively from day one.
4. Design alerts tied to those symptom-level metrics, with clear thresholds and
   a linked runbook or clear next action.
5. Build dashboards answering specific operational questions ("is the system
   healthy right now," "where's the latency coming from").
6. Periodically review alert noise (are pages actionable, are they being
   ignored) and prune/tune.

Decision gates:

- **What to log at what level**: `error` for things requiring attention,
  `warn` for unexpected-but-handled, `info` for significant business events,
  `debug` for detailed diagnostics not needed in normal production volume.
- **What to alert on**: user-facing symptoms (error rate, latency, availability)
  as high-severity pages; internal signals (queue depth trending up) as
  lower-severity or dashboard-only unless they reliably predict user impact.
- **Logs vs metrics vs traces**: metrics for aggregate trends and alerting;
  logs for detailed context on a specific event; traces for latency/behavior
  across a multi-component request path.
- **SLO targets**: set from actual user expectations and business impact, not
  an arbitrary "five nines" default.

Implementation rules:

- Propagate a correlation/request ID from entry through every log line,
  downstream call, and queue message.
- Use histograms for latency (percentiles, not averages — averages hide tail
  latency).
- Tag/label metrics with dimensions that matter (route, status code, service)
  without excessive cardinality.
- Every alert has a documented "what to do when this fires."
- Log enough context to diagnose without needing to reproduce — never log
  secrets, full PII payloads, or credentials.

## Solutions

Expected output: a concrete instrumentation plan or actual dashboard/alert
configuration, with explicit reasoning for what was included and left out.

Acceptance criteria:

- Every service has baseline observability: structured logs with correlation
  IDs, request rate/error rate/latency metrics, and health check status.
- Critical user-facing flows have explicit SLIs tracked.
- Alerting exists for user-facing symptom thresholds with documented response
  guidance.
- Telemetry access is access-controlled; sensitive data redacted.
- Telemetry pipeline degrades gracefully (logging/metrics failures don't crash
  requests); alerting has its own reliability.

## Runtime Constraints and Boundary Checks

- **NEVER**: log secrets, credentials, full auth tokens, or unredacted
  PII/payment data; create alerts with no actionable next step; average
  latency where percentiles are needed; claim a metric/alert "will catch" an
  issue without verifying it actually fires under the relevant condition.
- **STOP AND ASK when**: existing logging/metrics/tracing setup is unknown —
  inspect and build on established patterns before adding more.
- **STOP AND FLAG**: unstructured unsearchable logs, missing correlation-ID
  propagation across a boundary, alert fatigue, dashboards that don't answer a
  specific question, stale observability after the architecture changes.

## Interaction With Other Skills

- **backend-engineer / frontend-engineer**: this skill specifies what to
  instrument and how; they implement the instrumentation calls.
- **devops-engineer**: decides what telemetry is needed; devops-engineer
  provisions and operates collection/storage/shipping infra.
- **ai-engineer / rag-engineer / ai-agent-engineer**: applies the same
  principles to AI-specific telemetry (token usage, latency, retrieval
  quality, agent step traces).
- **security-engineer**: ensures telemetry access is controlled and doesn't
  leak sensitive data.
