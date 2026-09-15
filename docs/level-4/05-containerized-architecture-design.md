---
description: "Designing a Containerized Architecture — Everything so far has been about containerizing something already decided. This module is about the decision…"
---

# 05 · Designing a Containerized Architecture

Everything so far has been about containerizing something already
decided. This module is about the decision itself: how to decompose an
application into services, where to draw container boundaries, and how
to plan the runtime topology before writing a single Dockerfile.

## Decomposition: one process per container, grouped by change rate and scaling needs

The classic guideline "one process per container" is really shorthand
for two separate, more useful questions:

1. **Does this component need to scale independently?** A CPU-bound
   image-processing worker and a lightweight API gateway have wildly
   different resource profiles and load patterns — bundling them into one
   container forces them to scale together even though their actual
   demand curves don't match.
2. **Does this component change/deploy independently?** A component
   deployed weekly by one team shouldn't share a container (and thus a
   deployment unit) with one deployed multiple times a day by another —
   coupling their release cadence via a shared container creates
   unnecessary deployment risk and coordination overhead.

Components that share neither concern (they always scale together,
always deploy together, and are maintained by the same team) are
legitimate candidates to stay in one container despite the general
guideline — over-decomposing into many single-purpose containers
introduces coordination and network-hop overhead (module 04's
multi-container patterns) without a matching benefit.

## Defining boundaries: what crosses a container edge, and how

For each proposed boundary, decide explicitly:

- **Synchronous or asynchronous?** A direct HTTP/gRPC call couples the
  caller to the callee's availability in real time; a message queue
  (module 04's patterns extended with a broker container) decouples
  their uptime at the cost of eventual rather than immediate consistency.
- **Shared data or owned data?** Two services reading and writing the
  same database tables directly are not really separate services from a
  data-ownership standpoint — schema changes in one can silently break
  the other. Prefer each service owning its own data store (even if
  that's "just" its own schema within a shared Postgres instance) and
  exposing data to others only through its own API.
- **What's the network boundary's security posture?** Level 3's
  `internal: true` pattern generalizes here: draw network segments (public
  ingress, application tier, data tier) explicitly as part of the design,
  not as an afterthought once containers already exist.

## A worked example: decomposing a monolith

Starting point: one Rails/Django-style monolith doing web serving,
background job processing, and scheduled reports, all against one
Postgres database.

```
                    ┌─────────────┐
   Internet ───────▶│    proxy     │
                    └──────┬──────┘
                           │ public network
                    ┌──────▼──────┐        ┌─────────────┐
                    │   web-api    │───────▶│    redis     │ (job queue)
                    └──────┬──────┘        └──────┬──────┘
                           │ data network          │
                    ┌──────▼──────┐        ┌───────▼──────┐
                    │   postgres   │◀───────│    worker     │
                    └─────────────┘        └──────────────┘
                                                    ▲
                                            ┌───────┴──────┐
                                            │  scheduler    │ (cron-like, triggers jobs)
                                            └──────────────┘
```

`web-api` and `worker` are split because they scale on entirely different
signals (`web-api` on request rate, `worker` on queue depth) and because
a slow report job should never be able to starve web request handling by
competing for the same process's resources. `scheduler` is its own tiny
container specifically so its failure mode (missing a scheduled trigger)
is isolated from both — it doesn't serve traffic and doesn't process
jobs itself, only enqueues them.

```yaml
services:
  proxy: { image: nginx:1.25, networks: [public] }
  web-api:
    build: ./web
    networks: [public, data]
    deploy: { resources: { limits: { cpus: "1.0", memory: 512M } } }
  worker:
    build: ./worker
    networks: [data]
    deploy: { replicas: 3, resources: { limits: { cpus: "2.0", memory: "1G" } } }
  scheduler:
    build: ./scheduler
    networks: [data]
    deploy: { resources: { limits: { cpus: "0.1", memory: 64M } } }
  redis: { image: redis:7, networks: [data] }
  postgres: { image: postgres:16, networks: [data] }

networks:
  public: {}
  data: { internal: true }
```

## How It Actually Works

**Why independent scaling requirements translate directly into
independent resource limit blocks, mechanically.** As established in
Level 3, module 03, `deploy.resources.limits` writes directly to each
service's own cgroup. Two workloads sharing one container necessarily
share one cgroup and one process's resource ceiling — there is no way for
the kernel to grant "web request handling" more CPU than "report
generation" if both run inside the same container's cgroup, because the
kernel's CFS bandwidth controller and memory accounting operate at the
cgroup granularity, not the function-call granularity. Splitting them
into separate containers is therefore not merely an organizational
convenience — it's the only way the kernel's actual resource-isolation
primitives can be brought to bear independently on each workload.

**Why "owns its own data store" is enforceable at the network layer, not
just by convention.** A `data`-network service (Level 3's `internal:
true` topology) can be further partitioned so `worker` and `web-api`
each reach only the database schema/instance they're meant to own,
using Postgres's own per-role connection privileges plus, if warranted,
genuinely separate database containers per service rather than one
shared instance. Because network reachability is enforced by the kernel's
routing tables per network namespace (Level 3, module 01), a service with
no network path to another service's data store literally cannot bypass
its API to read that data directly, even via a bug or a compromised
dependency — the isolation holds regardless of what the application code
does or doesn't correctly enforce at the ORM/query layer.

## 🔀 Related lessons on other tracks

- [Kubernetes — 06 · Designing Production-Grade Cluster Architecture](https://sigilipelli.github.io/kubernetes-mastery-path/level-4/06-production-cluster-architecture/)
- [Terraform — 06 · Designing a Platform's Terraform Architecture](https://sigilipelli.github.io/terraform-mastery-path/level-4/06-platform-architecture/)

## Exercise

Take a hypothetical monolith you're familiar with (or the example above)
and produce a decomposition diagram identifying, for each proposed
service boundary: its independent scaling justification, whether it
communicates synchronously or asynchronously with its neighbors, which
network segment it belongs to, and which other service (if any) legitimately
owns the data it needs. Flag any boundary where you can't articulate a
concrete justification — that's a signal it may not need to be a separate
container at all.
