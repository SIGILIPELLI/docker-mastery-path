---
description: "Project — Multi-Container Compose App — This project pulls together everything from Level 2: multi-stage builds, Compose services/networks/volumes…"
---

# 10 · Project — Multi-Container Compose App

This project pulls together everything from Level 2: multi-stage builds,
Compose services/networks/volumes, environment configuration, health
checks, and image tagging, into one runnable application.

## The application

A small "link shortener" service: a Node.js API backed by Postgres for
persistent storage and Redis for a hit-count cache, fronted by an nginx
reverse proxy.

```
project/
├── docker-compose.yml
├── .env
├── nginx/
│   └── default.conf
└── api/
    ├── Dockerfile
    ├── package.json
    └── src/server.js
```

## The API's multi-stage Dockerfile

```dockerfile
# api/Dockerfile
FROM node:20 AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20-slim AS runtime
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NODE_ENV=production
EXPOSE 8000
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:8000/health',r=>process.exit(r.statusCode===200?0:1))"
CMD ["node", "src/server.js"]
```

## The reverse proxy config

```nginx
# nginx/default.conf
server {
    listen 80;
    location / {
        proxy_pass http://api:8000;
        proxy_set_header Host $host;
    }
}
```

## `.env`

```dotenv
POSTGRES_VERSION=16
REDIS_VERSION=7
NGINX_VERSION=1.25
APP_PORT=8080
```

## The full stack

```yaml
services:
  proxy:
    image: nginx:${NGINX_VERSION}
    ports:
      - "${APP_PORT}:80"
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - api
    networks: [public]

  api:
    build: ./api
    environment:
      - DATABASE_URL=postgres://app:devpassword@db:5432/links
      - REDIS_URL=redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started
    networks: [public, data]

  db:
    image: postgres:${POSTGRES_VERSION}
    environment:
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=devpassword
      - POSTGRES_DB=links
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 5
    networks: [data]

  cache:
    image: redis:${REDIS_VERSION}
    volumes:
      - redisdata:/data
    networks: [data]

networks:
  public:
  data:
    internal: true

volumes:
  pgdata:
  redisdata:
```

Note the topology from module 07: `proxy` cannot reach `db` or `cache`
directly (no shared network), `api` bridges `public` and `data`, and
`data` is `internal: true` so neither `db` nor `cache` can make outbound
connections even if compromised.

## Bringing it up and verifying

```bash
docker compose up -d --build
docker compose ps
# all four services Up, db and api showing (healthy)

curl -I http://localhost:8080/
# 200 via proxy -> api

docker compose logs -f api
docker compose exec proxy sh -c "getent hosts db" ; echo $?
# fails to resolve — confirms network isolation is actually enforced, not just declared

docker compose down
```

## Tagging and promoting the built image

```bash
docker compose build api
docker tag project-api:latest registry.example.com/team/link-shortener:1.0.0
docker push registry.example.com/team/link-shortener:1.0.0
```

## How It Actually Works

**Why this topology is a meaningfully stronger security posture than one
flat network, mechanically.** Each `networks:` entry here backs a real,
separate Linux bridge with its own subnet (module 07); `data` being
`internal: true` additionally means the daemon never creates the
`iptables` MASQUERADE/forwarding rules that would let containers on it
reach outside the bridge. Concretely, a request from a compromised `api`
container to some external attacker-controlled host still works (since
`api` is *also* on `public`, which isn't internal), but a compromised
`db` — which is *only* ever attached to `data` — has no route out at all:
its default gateway is the `data` bridge, and packets destined anywhere
off that bridge are simply dropped by the kernel's routing/netfilter
rules rather than forwarded, because no NAT rule exists to forward them.
This is defense-in-depth enforced by the kernel's networking stack, not
merely an application-level convention.

**Why the health-gated migration/startup ordering here reflects a real
distributed-systems concern, not just Compose bookkeeping.** Postgres's
own startup sequence includes crash-recovery replay from its
write-ahead log before it starts accepting connections — a period during
which the process is running (so a naive TCP-connect check might even
succeed against the listening socket) but not actually ready to serve
correct reads/writes. `pg_isready` calls Postgres's own internal
readiness-check RPC rather than merely testing whether *a* socket is
open, which is precisely why it's the correct health check to gate `api`
on, and why a generic "is port 5432 open" check would be a subtly wrong
substitute that could let `api` start querying a database still mid-recovery.

## Exercise

Add a fifth service, `backup`, using the official `postgres:16` image
(for its bundled `pg_dump` client) with no `ports:`, on the `data`
network only, that runs `pg_dump` against `db` on a schedule (a simple
`sh -c "while true; do pg_dump ...; sleep 86400; done"` is sufficient for
this exercise) and writes the dump to a new named volume,
`backups:/backups`. Confirm the dump file appears via
`docker compose exec backup ls /backups`.
