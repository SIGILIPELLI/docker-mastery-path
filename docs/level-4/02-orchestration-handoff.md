---
description: "Orchestration Handoff (Kubernetes/Swarm) — Swarm (Level 3, module 08) covers a real, useful slice of orchestration. This module is about recognizing the…"
---

# 02 · Orchestration Handoff (Kubernetes/Swarm)

Swarm (Level 3, module 08) covers a real, useful slice of orchestration.
This module is about recognizing the point where a workload has outgrown
plain Docker/Compose or Swarm and genuinely needs Kubernetes — and being
honest that not every project reaches that point.

## Signals it's time to move beyond Compose

- **More than a handful of hosts**, especially with heterogeneous
  hardware/scheduling constraints (GPU nodes, spot instances mixed with
  on-demand).
- **Multiple teams deploying independently** onto shared infrastructure,
  needing namespace-level isolation and RBAC rather than one shared
  Docker daemon.
- **Autoscaling based on real metrics** (request rate, queue depth), not
  just a fixed replica count.
- **A need for the broader ecosystem** — Helm charts for third-party
  software, operators that manage stateful services (databases, message
  queues) declaratively, service meshes for fine-grained traffic control.

## Signals Swarm (or plain Compose) is still the right size

- A single team, a handful of hosts, and predictable load — Swarm's
  operational simplicity (module 08: `docker swarm init`, no separate
  control-plane components to run) can be a genuine advantage, not a
  limitation, at this scale.
- The team has no existing Kubernetes operational expertise and the
  workload doesn't need anything Kubernetes uniquely provides — adopting
  Kubernetes purely for its reputation, without a concrete requirement it
  satisfies, tends to add operational burden without matching benefit.

## Concept mapping: Swarm to Kubernetes

| Swarm concept | Kubernetes equivalent | Key difference |
|---|---|---|
| Service (`docker service create`) | Deployment + Service | Kubernetes splits "desired replica set" (Deployment) from "stable network identity" (Service) into two objects |
| Task | Pod | A Pod can hold multiple co-scheduled containers sharing a network namespace (module 04 of Level 3's sidecar pattern, natively) |
| Overlay network | CNI network plugin | Kubernetes delegates networking to a pluggable CNI implementation rather than a single built-in driver |
| `docker secret` | Kubernetes `Secret` | Conceptually identical (mounted as files/env, module 09 of Level 3); Kubernetes secrets are base64-encoded by default, not encrypted, unless encryption-at-rest is separately configured |
| `docker config` | Kubernetes `ConfigMap` | Same purpose, same distinction from Secret |
| `docker stack deploy -c stack.yml` | `kubectl apply -f manifests/` | Both apply a declarative desired state; Kubernetes's object model is far larger (dozens of resource kinds vs. Compose's handful) |

## A Compose file's rough Kubernetes shape

The `stack.yml` from Level 3's capstone maps conceptually onto:

```yaml
# deployment.yaml (illustrative — not exhaustive)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: api
          image: registry.example.com/team/link-shortener:1.0.0
          resources:
            limits: { cpu: "1", memory: "256Mi" }
            requests: { cpu: "250m", memory: "128Mi" }
          volumeMounts:
            - name: db-secret
              mountPath: /run/secrets
      volumes:
        - name: db-secret
          secret: { secretName: database-url }
```

The `resources.limits`/`requests` block is the direct Kubernetes analog
of Swarm's `deploy.resources.limits`/`reservations` (Level 3, module 03's
cgroup mechanism underlies both identically — Kubernetes ultimately
configures the same kernel cgroups via its container runtime).

## Worked example: a migration decision, reasoned through

A team running the Level 3 capstone stack on 3 Swarm nodes starts needing
per-tenant namespace isolation for a new enterprise customer requirement,
autoscaling `api` based on request latency, and a managed Postgres
operator for automated failover. None of these are things Swarm provides
natively. That combination of concrete requirements — not vague
"Kubernetes is more mature" reasoning — is the actual trigger to migrate,
and the migration path is translating the existing `stack.yml` service by
service using the table above, rather than a rewrite from scratch.

## How It Actually Works

**Why Kubernetes needing a pluggable CNI, while Swarm ships one built-in
driver, reflects a real architectural difference, not just extra
options.** Swarm's overlay networking (Level 3, module 01) is baked
directly into the Docker Engine's own code — one implementation, VXLAN,
covering every Swarm cluster. Kubernetes instead defines the Container
Network Interface as a plugin contract (a binary the kubelet invokes with
a defined JSON input/output) precisely because different clusters have
genuinely different networking requirements — cloud-native clusters
often want a plugin that maps Pod IPs directly onto the cloud provider's
own VPC routing (avoiding VXLAN encapsulation overhead entirely), while
on-prem clusters commonly do need VXLAN or a similar overlay. This
pluggability is a direct consequence of Kubernetes targeting far more
heterogeneous deployment environments than Swarm was designed for.

**Why splitting "Deployment" and "Service" reflects a genuine
separation of concerns Swarm collapses into one object.** In Swarm, one
`docker service create` simultaneously declares the replica count *and*
the stable VIP/DNS name backing it. Kubernetes deliberately decouples
these: a Deployment only manages Pod replica lifecycle (create, replace,
roll out new versions) and has no inherent stable network identity at
all; a separate Service object provides the stable ClusterIP/DNS name and
uses a label selector to dynamically discover *whichever* Pods currently
match, regardless of which Deployment (or even multiple Deployments)
created them. This decoupling is what enables patterns Swarm can't
express directly, like a single Service load-balancing across two
different Deployments during a blue-green rollout, each independently
scaled.

## Exercise

Take the `stack.yml` from Level 3's capstone and, without necessarily
running it, write out the equivalent Kubernetes Deployment + Service +
Secret + NetworkPolicy YAML for just the `api` and `db` services,
explicitly noting where the `internal: true` overlay network's isolation
guarantee (Level 3, module 01/10) would need to be reproduced with a
Kubernetes NetworkPolicy instead, since Kubernetes has no direct
equivalent of "internal: true" on a network object itself.
