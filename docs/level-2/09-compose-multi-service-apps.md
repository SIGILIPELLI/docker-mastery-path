# 09 · Compose for Multi-Service Apps

A realistic application is rarely one container — it's a web tier, an
API, a database, and often a cache, wired together with the right start
order, environment, and network topology. This module puts modules 02–08
together into one coherent stack.

## A three-tier stack

```yaml
services:
  web:
    build: ./frontend
    ports:
      - "3000:3000"
    environment:
      - API_URL=http://api:8000
    depends_on:
      - api

  api:
    build: ./backend
    environment:
      - DATABASE_URL=postgres://app:devpassword@db:5432/appdb
      - REDIS_URL=redis://cache:6379
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      retries: 3
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_started

  db:
    image: postgres:16
    environment:
      - POSTGRES_USER=app
      - POSTGRES_PASSWORD=devpassword
      - POSTGRES_DB=appdb
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      retries: 5

  cache:
    image: redis:7
    volumes:
      - redisdata:/data

volumes:
  pgdata:
  redisdata:
```

## Why the dependency chain matters here

`web` depends on `api` being reachable at all (not necessarily fully
warmed up, since the frontend just proxies API calls); `api` genuinely
needs `db` to be accepting connections before it can run migrations or
serve requests, hence `condition: service_healthy` gated on Postgres's own
`pg_isready`. `cache` only needs `condition: service_started` — Redis
starts almost instantly and a missing cache is typically a
performance-degradation, not a hard-failure, condition for most apps
(design this per your own app's actual failure behavior).

## Scaling one service

```bash
docker compose up -d --scale api=3
```

Runs three containers for the `api` service simultaneously. This only
makes sense when `api` doesn't publish a fixed host port directly (three
containers can't all bind host port 8000) — typically you'd put a
load-balancing reverse proxy in front, or omit `ports:` on `api` and only
expose it internally, with `web`/the proxy reaching it by service name
(Compose's embedded DNS round-robins between the scaled instances' IPs).

## Overriding for local development

```yaml
# docker-compose.override.yml (loaded automatically alongside docker-compose.yml)
services:
  api:
    build:
      target: build     # use the debug-friendly stage from a multi-stage Dockerfile
    volumes:
      - ./backend:/app  # bind-mount source for live reload
    environment:
      - DEBUG=true
```

Compose automatically merges `docker-compose.override.yml` on top of
`docker-compose.yml` with no extra flags — this is the standard way to
keep a clean, production-shaped base file while layering
development-only conveniences (bind mounts, debug flags) separately, and
`.gitignore`-ing the override file if it contains machine-specific paths.

## Worked example: bringing the stack up and verifying it end-to-end

```bash
docker compose up -d --build
docker compose ps                    # confirm all four services report healthy/running
docker compose logs -f api           # watch api logs while it connects to db/cache
curl http://localhost:3000            # exercise the frontend, which calls through to api
docker compose down                  # stop everything, keep the named volumes
```

## How It Actually Works

**`depends_on` with a health condition is implemented as the daemon
polling, not the dependent container being paused mid-start.** When
`api` has `depends_on: db: condition: service_healthy`, Compose's control
plane (the `docker compose` CLI/engine, not the Docker Engine itself)
holds off issuing the `docker start` (or `create`+`start`) call for `api`
until it observes, via repeated `docker inspect`-equivalent API calls,
that `db`'s health check has reported `healthy` at least once. `api`'s
container doesn't exist yet at all during that wait — this is orchestration happening entirely in the Compose client/daemon coordination layer, before the dependent container is ever created, which is also why `depends_on` conditions have no effect on containers started outside Compose's `up` command (e.g. `docker start api` directly skips this check entirely).

**How scaled replicas share one service name.** When you run `--scale
api=3`, each replica is a distinct container (`myproject_api_1`,
`_2`, `_3`) individually attached to the project's network with its own
IP, but all three register the *same* DNS alias (`api`) with the
embedded DNS server described in module 02. A lookup for `api` from
another container returns all three IPs (or one, chosen by the resolver,
depending on configuration) — this is why scaling works transparently for
service-name-based discovery without any load balancer configuration,
but also why it doesn't work at all for anything relying on a single
fixed host port, since three processes can't share one host-side listen
socket.

## Exercise

Extend the stack above with a fifth service, `migrate`, built from the
same image as `api` but running a one-off migration command instead of
the server, with `depends_on: db: condition: service_healthy` and no
`restart` policy (so it runs once and exits cleanly with code 0). Make
`api` additionally depend on `migrate` completing successfully before it
starts — research Compose's `condition: service_completed_successfully`
for this exact use case.
