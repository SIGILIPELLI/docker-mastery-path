---
description: "Observability for Containers — Logs (Level 3, module 07) are one of three observability pillars. Metrics and traces fill in what logs alone can't…"
---

# 06 · Observability for Containers

Logs (Level 3, module 07) are one of three observability pillars. Metrics
and traces fill in what logs alone can't: aggregate trends over time, and
the causal path of one request across several containers.

## Metrics: exposing and scraping them

The dominant pattern is exposing an HTTP endpoint that a metrics
collector scrapes periodically, rather than the app pushing metrics
itself:

```dockerfile
# app exposes /metrics in Prometheus text format on its own port
EXPOSE 8000
```

```yaml
services:
  api:
    build: .
    ports: ["8000:8000"]

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports: ["9090:9090"]
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: api
    static_configs:
      - targets: ["api:8000"]
```

Prometheus's embedded DNS resolution of `api` (Level 2, module 02's
service-name discovery) works identically here — the metrics collector is
just another container on the same Compose network reaching `api` by
service name.

## `cAdvisor` for container-level resource metrics without app changes

```yaml
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    ports: ["8080:8080"]
```

cAdvisor exposes per-container CPU/memory/network/disk metrics (the same
underlying cgroup data `docker stats`, Level 3 module 03, reads) in
Prometheus format automatically, without any instrumentation inside the
application containers themselves — useful for infrastructure-level
dashboards even for services you don't control the code of.

## Distributed tracing across container boundaries

A single user request that crosses `proxy` → `api` → `worker` → `db`
(module 05's decomposed architecture) needs a trace ID that survives each
hop to reconstruct the full path and per-hop latency:

```yaml
services:
  api:
    build: .
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_SERVICE_NAME=api

  worker:
    build: ./worker
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_SERVICE_NAME=worker

  otel-collector:
    image: otel/opentelemetry-collector:latest
    volumes:
      - ./otel-collector-config.yml:/etc/otelcol/config.yaml:ro
```

Each service's OpenTelemetry SDK propagates a trace context (via HTTP
headers on synchronous calls, or message attributes on queue-based calls)
so spans from every container involved in one request get correlated
under one trace ID when they arrive at `otel-collector`, which then
exports them onward to a backend (Jaeger, Tempo, a vendor's APM).

## Worked example: correlating a slow request across the stack

1. A user reports a slow page load.
2. The trace UI shows one trace: 40ms in `proxy`, 15ms in `api`, then a
   380ms span in `worker` waiting on a `db` query.
3. Cross-reference that time window in Prometheus: `db`'s connection pool
   utilization metric spiked to 100% at that exact timestamp.
4. Cross-reference container-level metrics from cAdvisor: `db`'s
   container CPU was pegged at its `--cpus` limit (Level 3, module 03)
   during that window — the query wasn't inherently slow, the database
   container was CPU-throttled.

This is the concrete value of having all three pillars: logs alone would
show a slow query with no obvious cause; metrics alone would show CPU
pressure with no obvious *request* impact; tracing alone would show which
hop was slow but not why. Together they answer both "where" and "why" in
one investigation.

## How It Actually Works

**Why a pull-based scrape model (Prometheus) behaves differently under
container churn than a push-based one.** Prometheus's scraper connects to
each target's `/metrics` endpoint on a fixed interval using the *current*
resolution of that target's address — for a short-lived or restarted
container (a new IP behind the same service-name alias, per Level 2
module 02's embedded DNS), the next scrape simply resolves to wherever
that alias currently points. A push-based system, in contrast, requires
the *application* to know the collector's address and push proactively —
which fails silently if a container starts and dies faster than its
first push interval, a real risk for the kind of very-short-lived batch
containers common in some architectures. This is why Prometheus's
ecosystem includes a "Pushgateway" specifically as an exception path for
jobs too short-lived to be reliably scraped.

**Why trace context propagation has to travel inside the request/message
itself, not alongside it.** A trace ID and its parent span ID are
injected into the outgoing HTTP request's headers (commonly the
`traceparent` header, per the W3C Trace Context standard) precisely
because container-to-container calls in a distributed system have no
other shared, ordered channel connecting them — unlike function calls
within a single process, which share a call stack the tracer can walk
directly. Each service's OpenTelemetry instrumentation, on receiving a
request, extracts that header, creates a new child span linked to the
same trace ID, and re-injects an updated header on any further outbound
calls it makes — the trace is reconstructed after the fact purely by the
collector correlating every span that shares a trace ID, since no single
component ever has visibility into the whole path in real time.

## Exercise

Add a Prometheus + cAdvisor pair to any Compose stack from an earlier
module, confirm cAdvisor reports per-container CPU/memory matching what
`docker stats` shows for the same container, then deliberately impose a
tight `--cpus` limit on one service and confirm the resulting throttling
is visible as a metric in Prometheus's UI, not just anecdotally via
"the app feels slow."
