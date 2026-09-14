# 02 · Docker Compose Basics

**Compose** lets you describe a multi-container application declaratively
in a YAML file and bring the whole thing up or down with one command,
instead of chaining together long `docker run`, `docker network create`,
and `docker volume create` invocations by hand.

## A minimal `docker-compose.yml`

```yaml
services:
  web:
    build: .
    ports:
      - "8000:8000"
    environment:
      - PYTHONUNBUFFERED=1
    depends_on:
      - db
  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=devpassword
    volumes:
      - db-data:/var/lib/postgresql/data

volumes:
  db-data:
```

```bash
docker compose up -d       # build (if needed) and start both services
docker compose ps          # list services and their status
docker compose logs -f web # follow logs for one service
docker compose down        # stop and remove containers, network (keeps named volumes)
```

## What each top-level key does

| Key | Purpose |
|---|---|
| `services` | One entry per container Compose manages; the key (`web`, `db`) becomes both the container's name prefix and its hostname on the Compose network |
| `build` | Build an image from a local Dockerfile instead of pulling one |
| `image` | Pull (or reference) an existing image by name/tag |
| `ports` | `HOST:CONTAINER` port publishing, same semantics as `docker run -p` |
| `environment` | Environment variables injected into the container |
| `volumes` (service-level) | Bind mounts or named-volume mounts for that service |
| `volumes` (top-level) | Declares named volumes so Compose knows to create/manage them |
| `depends_on` | Controls **start order** only — it does not wait for the dependency to be *ready* (see module 06 for that) |
| `networks` | Custom networks beyond the default one Compose creates automatically |

## Worked example: build + run in one step

```bash
docker compose up -d --build
```

`--build` forces Compose to rebuild the `web` image from its Dockerfile
before starting, which matters after you've changed source code or the
Dockerfile itself — `docker compose up` alone reuses a previously built
image if one already exists with that name.

```bash
docker compose exec web sh -c "python manage.py migrate"
```

`docker compose exec` runs a one-off command inside an already-running
service container, analogous to `docker exec` but addressed by service
name instead of container ID.

## The implicit network and service discovery

Compose creates one default bridge network per project (named after the
project directory) and attaches every service to it. Inside that network,
each service is reachable by its service name as a DNS hostname — so in
the example above, `web`'s code connects to the database at host `db`,
port `5432`, not `localhost`. This works with zero explicit `networks:`
configuration.

## How It Actually Works

**Compose is a thin orchestration layer over the same Docker Engine
primitives you'd use manually.** `docker compose up` on the example above
is functionally equivalent to: `docker network create <project>_default`,
`docker volume create <project>_db-data`, then `docker run` for `db` and
`web` each attached (`--network <project>_default`) with their declared
ports/env/volumes, plus each container registered under its service name
as a network alias. Nothing about Compose's runtime behavior is special
to Compose itself — it is generating and sequencing ordinary Engine API
calls, which is why anything you can do with `docker run` flags has a
corresponding YAML key, and why `docker inspect` on a Compose-managed
container looks just like any other container's inspect output (with
`com.docker.compose.*` labels added for bookkeeping).

**Service-name DNS resolution.** The Docker Engine runs an embedded DNS
server (at `127.0.0.11` inside each container's network namespace) for
every user-defined bridge network. When a container looks up hostname
`db`, its `/etc/resolv.conf` (rewritten by the Engine at container start)
points resolution at `127.0.0.11`, which answers with the current IP of
whatever container is attached to that network under the alias `db` —
resolved dynamically from the Engine's internal network-to-container
mapping, not baked into `/etc/hosts` as a static entry (though a static
entry is *also* written for the container's own name, for compatibility).
This is why service discovery keeps working even if `db`'s container is
recreated with a new IP: the alias, not the IP, is what other containers
resolve.

## Exercise

Write a `docker-compose.yml` for a two-service app: a `redis:7` cache
service (no ports published, only reachable internally) and a `worker`
service built from a local Dockerfile that connects to it at hostname
`redis`, port `6379`. Bring it up with `docker compose up -d`, confirm
with `docker compose exec worker sh -c "getent hosts redis"` that the
hostname resolves, then tear it down with `docker compose down -v`.
