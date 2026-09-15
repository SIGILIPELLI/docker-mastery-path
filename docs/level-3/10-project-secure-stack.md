---
description: "Project — Secure, Resource-Limited Production Stack — This capstone for Level 3 combines security hardening, resource limits, multi-container patterns…"
---

# 10 · Project — Secure, Resource-Limited Production Stack

This capstone for Level 3 combines security hardening, resource limits,
multi-container patterns, private registry usage, centralized logging,
and secrets into one deployable stack — everything from modules 01–09.

## The application

The same link-shortener from Level 2's project, hardened for production:
a non-root, resource-capped API; Postgres with a managed secret instead
of a plain-text password; an nginx proxy; and a log-shipping sidecar.

```
project/
├── stack.yml
├── nginx/default.conf
├── secrets/db_password.txt   (gitignored; created via docker secret in Swarm)
└── api/Dockerfile
```

## Hardened API Dockerfile

```dockerfile
# api/Dockerfile
FROM node:20 AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:20-slim
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY --chown=node:node . .
USER node
ENV NODE_ENV=production
EXPOSE 8000
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8000/health',r=>process.exit(r.statusCode===200?0:1))"
CMD ["node", "src/server.js"]
```

## The stack file

```yaml
# stack.yml
version: "3.9"

services:
  proxy:
    image: nginx:1.25
    ports:
      - "8080:80"
    configs:
      - source: nginx_conf
        target: /etc/nginx/conf.d/default.conf
    networks: [public]
    deploy:
      resources:
        limits:
          cpus: "0.5"
          memory: 128M

  api:
    image: registry.example.com/team/link-shortener:1.0.0
    environment:
      - DATABASE_URL_FILE=/run/secrets/database_url
    secrets:
      - database_url
    networks: [public, data]
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: "1.0"
          memory: 256M
        reservations:
          cpus: "0.25"
          memory: 128M
      restart_policy:
        condition: on-failure
      update_config:
        parallelism: 1
        delay: 10s

  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
      - POSTGRES_USER=app
      - POSTGRES_DB=links
    secrets:
      - db_password
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks: [data]
    deploy:
      resources:
        limits:
          memory: 512M

  log-shipper:
    image: fluent/fluent-bit:latest
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    networks: [public]
    deploy:
      resources:
        limits:
          memory: 64M

networks:
  public:
  data:
    driver: overlay
    internal: true

volumes:
  pgdata:

secrets:
  db_password:
    external: true
  database_url:
    external: true

configs:
  nginx_conf:
    file: ./nginx/default.conf
```

## Deploying it

```bash
docker swarm init   # if not already a Swarm

echo "supersecretpassword" | docker secret create db_password -
echo "postgres://app:supersecretpassword@db:5432/links" | docker secret create database_url -

docker stack deploy -c stack.yml linkshortener
docker stack services linkshortener
docker service ps linkshortener_api
```

## Verifying the hardening actually holds

```bash
# api runs as non-root
docker exec $(docker ps -q -f name=linkshortener_api) whoami
# node, not root

# secret is file-mounted, not an env var
docker exec $(docker ps -q -f name=linkshortener_api) env | grep -i database_url
# nothing — the real DATABASE_URL_FILE points at a mounted file, not an inline value

# data network cannot reach outside
docker exec $(docker ps -q -f name=linkshortener_db) sh -c "wget -T 3 -qO- https://example.com" ; echo $?
# fails — data network is internal: true

# resource limits are enforced
docker inspect $(docker ps -q -f name=linkshortener_api) -f '{{.HostConfig.Memory}}'
# 268435456 (256MB in bytes)
```

## How It Actually Works

**Why the rolling update (`update_config`) matters specifically for a
service holding a database connection secret.** When `api`'s image is
updated, Swarm's rolling update (module 08's reconciliation loop, applied
one task at a time per `parallelism: 1`) starts a new task alongside the
old ones before stopping any, waits for it to report healthy via the
`HEALTHCHECK` before proceeding to the next, and never removes more
capacity than it has already replaced. Combined with `db`'s `secrets`
being mounted identically for every replica by name (not baked into any
specific image build), a secret rotation and an image update can proceed
independently: rotating `database_url` (module 09's `--secret-add`
pattern) doesn't require rebuilding the `api` image, and updating the
`api` image doesn't require touching the secret.

**Why the layered network topology (`public`/`data internal: true`)
plus per-service memory limits together bound the blast radius of a
single compromised container more effectively than either alone.** A
resource limit constrains what a compromised process can *consume*
(module 03's cgroup enforcement) but says nothing about what it can
*reach*; network isolation constrains what it can reach (module 01's
routing/NAT enforcement) but says nothing about resource exhaustion. A
compromised `api` replica, memory-capped and only able to reach `public`
and `data`, cannot exhaust the host's total memory (kernel OOM killer
scoped to its own cgroup, per module 03) and cannot pivot to attack
anything outside those two networks (no route exists, per module 01) —
each control independently closes a different failure mode, which is why
production hardening treats them as complementary layers rather than
picking one.

## Exercise

Add a `--limit-cpu`/`--limit-memory` breach test: temporarily lower
`api`'s memory limit to something unreasonably small (e.g. `32M`), redeploy
the stack, and observe via `docker service ps` and `docker service logs`
that replicas now OOM-kill and restart in a loop (module 03's `137` exit
code, module 08's automatic rescheduling) — then restore the original
limit and confirm the service stabilizes.
