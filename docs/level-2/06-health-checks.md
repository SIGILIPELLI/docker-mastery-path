# 06 · Health Checks

`docker ps` showing `Up` only means the main process hasn't exited — it
says nothing about whether the app inside is actually able to serve
traffic. A **health check** runs a command periodically inside the
container and reports `healthy`/`unhealthy`/`starting`, giving tooling a
real readiness signal.

## `HEALTHCHECK` in a Dockerfile

```dockerfile
FROM nginx:1.25
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost/ || exit 1
```

| Option | Meaning |
|---|---|
| `--interval` | Time between checks (default 30s) |
| `--timeout` | How long a single check may run before it's considered failed |
| `--start-period` | Grace period after container start during which failures don't count toward `--retries` (for slow-starting apps) |
| `--retries` | Consecutive failures required before marking `unhealthy` |

The check command must exit `0` for healthy, `1` for unhealthy — exactly
like any shell command's exit code convention. `curl -f` (fail on HTTP
error status) is a common check for HTTP services; for services without
`curl` installed, a small script using the runtime's own HTTP client
(e.g. Python's `urllib`) avoids adding a dependency just for this.

## Observing health status

```bash
docker ps
# STATUS column shows "Up 2 minutes (healthy)" or "(unhealthy)"

docker inspect -f '{{.State.Health.Status}}' myapp
docker inspect -f '{{json .State.Health.Log}}' myapp | python3 -m json.tool
```

`.State.Health.Log` keeps the last several check results (output,
duration, exit code) — the first place to look when a container is stuck
`unhealthy` and it isn't obvious why.

## Health checks in Compose

```yaml
services:
  api:
    build: .
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 10s
      timeout: 3s
      start_period: 5s
      retries: 3

  worker:
    build: ./worker
    depends_on:
      api:
        condition: service_healthy
```

`depends_on` with `condition: service_healthy` is what turns "start
order" into "actual readiness" — `worker` won't start until `api`'s
health check reports healthy, not merely running. Without a health check,
`depends_on` only guarantees `api`'s container process has begun
starting, which for anything with its own startup time (loading models,
running migrations, warming a cache) is not the same as being ready to
serve requests.

## Worked example: a database-dependent service

```yaml
services:
  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=devpassword
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 3s
      retries: 5

  web:
    build: .
    depends_on:
      db:
        condition: service_healthy
```

`pg_isready` is Postgres's own bundled readiness-probe binary — preferring
a service's own health-check tool (over guessing with a raw TCP connect)
avoids false "healthy" reports from a database that has accepted a TCP
connection but hasn't finished recovery/initialization yet.

## How It Actually Works

**The health check runs as its own short-lived process inside the
container's namespaces, on a timer owned by the daemon.** The Docker
daemon maintains a per-container timer; on each tick it uses the same
`exec`-into-container mechanism as `docker exec` (joining the container's
namespaces via `setns()`) to run the check command, capturing its exit
code, stdout+stderr (truncated), and duration into
`.State.Health.Log`. This is why a healthcheck sees exactly what
`docker exec` would see — the container's own filesystem, network stack,
and installed binaries — and why a `HEALTHCHECK` command that hangs
matters: it doesn't get killed automatically at `--timeout`'s natural
end without daemon involvement, but rather the daemon marks that
individual check attempt as failed once `--timeout` elapses and moves on,
which is why a check command should itself be one that returns quickly
rather than one that can block indefinitely.

**Why `--start-period` exists as a genuinely separate concept from
`--interval`.** Without it, a slow-starting service (JVM warm-up, model
loading) would rack up `--retries` consecutive failures before it's ever
had a fair chance to become ready, tripping `unhealthy` even though
nothing is actually wrong. The daemon tracks two separate counters:
elapsed wall-clock time since container start (compared against
`--start-period`) and a consecutive-failure count (compared against
`--retries`) — failures during the start period still run and still get
logged, but they don't increment the counter that can flip the container
to `unhealthy`. Once `--start-period` has elapsed, ordinary
`--interval`/`--retries` accounting takes over for the rest of the
container's life, including any future failures after a period of being
healthy.

## Exercise

Add a `HEALTHCHECK` to a Dockerfile for a simple HTTP service (any
language) that checks `GET /` every 5 seconds with a 2-second timeout and
3 retries. Deliberately make the app sleep for 8 seconds before binding
its port on startup, and use `--start-period` correctly so the container
doesn't flip to `unhealthy` during that startup window. Verify with
`docker inspect -f '{{json .State.Health}}'`.
