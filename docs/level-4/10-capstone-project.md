---
description: "Capstone Project — This capstone pulls together the entire path — Dockerfiles (Level 1), Compose and multi-container patterns (Level 2/3), orchestration…"
---

# 10 · Capstone Project

This capstone pulls together the entire path — Dockerfiles (Level 1),
Compose and multi-container patterns (Level 2/3), orchestration and
hardening (Level 4 modules 01-06), and this level's deployment, cost, and
recovery practices (modules 07-09) — into one designed and documented
system. There's no new syntax here; the goal is putting it all together
correctly, and writing down *why*, the way a real production handoff
document would.

## The system to design

A small but realistically-shaped web application:

- `web` — a stateless API service (the kind of thing hardened in module
  03 and scaled in module 04's build-optimization sense).
- `db` — a stateful Postgres service backed by a named volume.
- `cache` — Redis, also stateful, also a named volume.
- A reverse proxy in front of `web` for TLS termination and routing.

## Step 1: multi-stage, hardened Dockerfile (modules 03-04)

```dockerfile
FROM node:20-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-slim
RUN groupadd -r app && useradd -r -g app app
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY . .
USER app
HEALTHCHECK --interval=5s --timeout=3s --start-period=10s --retries=3 \
  CMD node healthcheck.js || exit 1
EXPOSE 3000
CMD ["node", "server.js"]
```

Non-root `USER`, a multi-stage build that never ships dev dependencies,
and a real health check are the module 03/04/07 baseline for anything
going to production — not optional extras.

## Step 2: the full stack, with resource limits (module 08) and secrets (module 09, Level 3)

```yaml
# docker-compose.yml
version: "3.9"
services:
  proxy:
    image: nginx:1.25-alpine
    ports:
      - "443:443"
    configs:
      - source: nginx_conf
        target: /etc/nginx/nginx.conf
    depends_on:
      - web
    deploy:
      resources:
        limits: { cpus: "0.25", memory: 64M }

  web:
    build: .
    deploy:
      replicas: 3
      resources:
        limits: { cpus: "0.5", memory: 200M }
        reservations: { cpus: "0.15", memory: 128M }
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
        failure_action: rollback
    environment:
      - DATABASE_URL_FILE=/run/secrets/db_url
    secrets:
      - db_url
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    secrets:
      - db_url
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 5
    deploy:
      resources:
        limits: { cpus: "1.0", memory: 512M }

  cache:
    image: redis:7-alpine
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 5
    deploy:
      resources:
        limits: { cpus: "0.25", memory: 128M }

volumes:
  pgdata:
  redisdata:

secrets:
  db_url:
    external: true

configs:
  nginx_conf:
    file: ./nginx.conf
```

Every stateful service (`db`, `cache`) gets a health check gating
`depends_on: condition: service_healthy` on `web` — `web` never starts
accepting requests before its dependencies are actually ready, not merely
"container running."

## Step 3: CI/CD pipeline (module 01)

```yaml
# .github/workflows/deploy.yml (excerpt)
- name: Build and push
  run: |
    docker build -t registry.example.com/myapp/web:${{ github.sha }} .
    docker push registry.example.com/myapp/web:${{ github.sha }}
- name: Deploy with rolling update
  run: |
    docker service update \
      --image registry.example.com/myapp/web:${{ github.sha }} \
      --update-order start-first \
      --update-failure-action rollback \
      myapp_web
```

## Step 4: backup schedule (module 09)

```bash
# cron, daily
0 2 * * * docker exec myapp_db pg_dump -U postgres mydb | gzip > /backups/mydb-$(date +\%Y\%m\%d).sql.gz
```

## Step 5: the architecture write-up

A real handoff includes a short document alongside the compose file
covering:

- **Topology** — which services talk to which, and over what network
  (module 01's networking, module 05's design principles).
- **Failure modes** — what happens if `db` goes down (module 09's
  recovery plan), if a bad `web` image ships (module 07's rollback), if a
  host runs out of capacity (module 08's sizing).
- **Observability** — what module 06's logging/metrics setup would surface
  first when something breaks, and where an operator looks.

## Worked example: proving the whole thing survives a bad deploy

```bash
docker stack deploy -c docker-compose.yml myapp
docker service update --image registry.example.com/myapp/web:broken myapp_web
docker service ps myapp_web
# watch failure_action: rollback revert automatically -- module 07's
# mechanism, now exercised inside the full stack rather than in isolation
docker exec myapp_db pg_isready -U postgres
# confirm the database was never touched by the bad web deploy at all --
# the stateless/stateful separation (module 05) means a broken API image
# has no path to corrupt data even during a failed rollout
```

## How It Actually Works

**Why stateless/stateful separation is what makes every other module's
safety mechanism actually work together, mechanistically.** Rolling
updates with rollback (module 07) are safe specifically *because*
`web`'s containers carry no data of their own — replacing, killing, or
rolling back a `web` task destroys nothing that isn't trivially
reconstructible from the image, so the blast radius of a bad deploy is
capped at "temporary reduced capacity," never "lost data." `db` and
`cache`, by contrast, are excluded from that same rolling-replacement
treatment (they run as singleton or carefully-coordinated replicas backed
by named volumes, not freely interchangeable tasks) precisely because
their disks hold the one copy of state that a rollback cannot regenerate.
The architecture's resilience isn't one mechanism — it's this boundary
placed correctly, so that each subsystem gets the failure-handling
strategy that actually matches whether it holds recoverable state or not.

**Why `depends_on: condition: service_healthy` is a real ordering
guarantee and not just documentation.** The container runtime's default
`depends_on` (start order only) guarantees nothing about the dependency
being *usable*, because "container process started" and "database ready
to accept connections" are different points in time separated by however
long Postgres takes to run recovery and reach `pg_isready`. The
`service_healthy` condition ties startup ordering to the same
`HEALTHCHECK` state machine covered in module 07 — Compose/Swarm polls
the dependency's health status and withholds starting the dependent
service's containers until that state machine reports `healthy`, which is
the same readiness gate used for zero-downtime rolling updates, now
reused for the orthogonal purpose of startup sequencing. One mechanism,
two jobs: don't route traffic to an unready replica, and don't start a
consumer before its dependency is truly serving.

## Exercise

Build the four-service stack described above (a stub `web` app that reads
`DATABASE_URL_FILE` and responds to `/healthz` is enough — the plumbing
matters more than the business logic), deploy it, then run the "bad
deploy" worked example against your own stack and confirm two things
independently: that `myapp_web`'s replicas roll back automatically, and
that `pg_isready` against `db` never reports a gap in availability at any
point during the failed rollout.
