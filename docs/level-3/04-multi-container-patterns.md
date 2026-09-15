---
description: "Multi-Container App Patterns — Beyond 'one container per service,' a few recurring patterns compose containers to solve problems that don't fit neatly…"
---

# 04 · Multi-Container App Patterns

Beyond "one container per service," a few recurring patterns compose
containers to solve problems that don't fit neatly inside a single
process: augmenting a main container without modifying it, translating
between incompatible interfaces, and doing setup work before the main
container starts.

## Sidecar pattern

A **sidecar** runs alongside a main container, sharing its network
namespace (and often a volume), adding a capability without changing the
main container's image — log shipping, a proxy, a metrics exporter.

```yaml
services:
  app:
    build: .
    volumes:
      - applogs:/var/log/app

  log-shipper:
    image: fluent/fluent-bit:latest
    volumes:
      - applogs:/var/log/app:ro
    depends_on:
      - app

volumes:
  applogs:
```

`app` writes logs to a shared volume; `log-shipper` reads them read-only
and forwards them to an external aggregator — `app`'s image never needed
to know fluent-bit exists.

## Ambassador pattern

An **ambassador** is a small proxy container that sits between your
application and an external dependency, so the app always talks to a
stable local address while the ambassador handles the real, possibly
changing, endpoint.

```yaml
services:
  app:
    build: .
    environment:
      - DB_HOST=db-ambassador
      - DB_PORT=5432

  db-ambassador:
    image: haproxy:2.9
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
```

```
# haproxy.cfg (excerpt)
frontend db_front
    bind *:5432
    default_backend db_actual
backend db_actual
    server db1 real-db-host-1:5432 check
    server db2 real-db-host-2:5432 check backup
```

If the real database's address changes (failover to a replica, migration
to a new host), only `haproxy.cfg` changes — `app`'s configuration
(`DB_HOST=db-ambassador`) never does.

## Init-container pattern

An **init container** runs to completion *before* the main container
starts, performing setup the main container depends on — schema
migrations, waiting for a dependency, fetching a config file.

```yaml
services:
  migrate:
    build: .
    command: ["npm", "run", "migrate"]
    depends_on:
      db:
        condition: service_healthy
    restart: "no"

  api:
    build: .
    command: ["node", "src/server.js"]
    depends_on:
      migrate:
        condition: service_completed_successfully
      db:
        condition: service_healthy

  db:
    image: postgres:16
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
```

`condition: service_completed_successfully` (Compose's direct equivalent
of Kubernetes's init-container concept) blocks `api` from starting until
`migrate` has exited with code 0 — a failed migration (nonzero exit)
correctly prevents `api` from ever starting against an unmigrated schema.

## Worked example: combining sidecar and init-container

```yaml
services:
  warm-cache:
    build: ./cache-warmer
    depends_on:
      cache:
        condition: service_started
    restart: "no"

  app:
    build: .
    depends_on:
      warm-cache:
        condition: service_completed_successfully
    volumes:
      - metrics:/var/run/metrics

  metrics-sidecar:
    image: prom/statsd-exporter
    volumes:
      - metrics:/var/run/metrics:ro

  cache:
    image: redis:7

volumes:
  metrics:
```

`warm-cache` pre-populates Redis before `app` starts serving traffic
(init-container role); `metrics-sidecar` continuously exports whatever
`app` writes to the shared `metrics` volume (sidecar role) — two
different lifecycles, two different patterns, in one stack.

## How It Actually Works

**Why sidecars sharing "the same network namespace" is stronger than
sharing a Compose network.** In Kubernetes, sidecar containers within one
Pod literally share a single network namespace — `localhost` inside one
container reaches a port bound in the other, because there is only one
network stack for the whole Pod. Plain Docker/Compose containers, in
contrast, each get their own network namespace even on the same Compose
network — they reach each other via their own IP addresses over the
bridge, not via `localhost`. Docker does support the tighter,
Kubernetes-Pod-like model directly: `docker run --network
container:other_container_name` joins a *new* container to an *existing*
container's network namespace rather than creating its own, which is
the actual mechanism (`--pid=container:X` does the same for PID
namespaces) — worth knowing when a sidecar genuinely needs
`localhost`-level coupling rather than network-level coupling.

**Why `service_completed_successfully` is a real distinct state, not
just "container gone."** Compose tracks a container's exit code the same
way `docker inspect -f '{{.State.ExitCode}}'` would — a container that
exits 0 (success) and one that exits 1 (a crashed migration) both leave
the container in a `Exited` state indistinguishable by mere presence or
absence, so Compose specifically checks the recorded exit code before
satisfying that dependency condition. This is the same underlying signal
covered in the lifecycle module (Level 1, module 07) and the OOM
discussion (module 03 here) — Compose's dependency graph is, at bottom,
just automated polling of the same `State.ExitCode`/`State.Health`
fields you can inspect manually.

## Exercise

Build an init-container pattern where a `wait-for-db` service uses
`pg_isready` in a retry loop against `db` (rather than relying on
`db`'s own health check) and exits 0 only once it succeeds, and an `app`
service that depends on `wait-for-db` completing successfully. Then
break it deliberately — misconfigure `db`'s credentials so `wait-for-db`
never succeeds — and confirm `app` correctly never starts.
