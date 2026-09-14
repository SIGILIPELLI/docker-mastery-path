# 07 · Compose Networks & Volumes

Beyond the default network Compose creates automatically, you can declare
**named networks** and **named volumes** explicitly for finer control
over which services can reach each other and where persistent data lives.

## Why declare networks explicitly

By default every service in a `docker-compose.yml` joins one shared
network, meaning every service can reach every other service. For a stack
with, say, a public web tier and an internal database, that's more access
than necessary — a compromised web container shouldn't be able to see
services it has no business talking to.

```yaml
services:
  web:
    build: .
    networks:
      - frontend
      - backend
  db:
    image: postgres:16
    networks:
      - backend
  cache:
    image: redis:7
    networks:
      - backend

networks:
  frontend:
  backend:
```

`web` sits on both networks and can reach `db`/`cache`; nothing on
`frontend` alone could reach `db` or `cache`, and — if there were a public
proxy service on `frontend` only — it would have no route to the database
at all, since they don't share a network.

## Named volumes vs. bind mounts, declared in Compose

```yaml
services:
  db:
    image: postgres:16
    volumes:
      - db-data:/var/lib/postgresql/data       # named volume: Docker-managed
      - ./init-scripts:/docker-entrypoint-initdb.d  # bind mount: host path

volumes:
  db-data:
```

A **named volume** (`db-data`) is created and managed by Docker itself
(under `/var/lib/docker/volumes/` on Linux) — it persists across
`docker compose down` and container recreation, and its lifecycle is
independent of any specific container. A **bind mount** (`./init-scripts`)
maps a literal host path into the container — useful for injecting files
you edit on the host (init scripts, local source code for live-reload
during development) but tying the container to that host's filesystem
layout.

## Marking a volume external

```yaml
volumes:
  db-data:
    external: true
```

`external: true` tells Compose the volume already exists (created outside
this project, or by a previous `docker compose up`) and it must not try
to create or manage its lifecycle — `docker compose down -v` will refuse
to remove it. This matters when a volume holds production data that must
outlive any particular Compose project's teardown.

## Worked example: isolated tiers with a shared cache

```yaml
services:
  api:
    build: .
    networks: [public, data]
  admin:
    build: ./admin
    networks: [public, data]
  db:
    image: postgres:16
    networks: [data]
    volumes:
      - pgdata:/var/lib/postgresql/data

networks:
  public:
  data:
    internal: true   # no route to the outside world at all, even for containers on it

volumes:
  pgdata:
```

`internal: true` on the `data` network additionally blocks that network
from routing to the outside internet (not just isolating it from other
Compose networks) — appropriate for a database tier that has no business
making outbound connections.

## How It Actually Works

**Each Compose network is an ordinary Docker bridge network under the
hood.** `docker network ls` after `docker compose up` shows
`<project>_frontend` and `<project>_backend` as real bridge networks,
each getting its own Linux bridge interface and IP subnet from the
daemon's address-pool allocator, with veth pairs connecting each attached
container's network namespace to that bridge. A service on two networks
(like `web` above) simply has two veth pairs and two IP addresses — one
per bridge — and the kernel's normal routing table inside that
container's network namespace decides which interface a given
destination IP routes through. There's no Compose-specific network
technology involved; "networks:" is purely a declarative way of driving
`docker network create` and `docker network connect` for you in the
right order.

**Why a named volume survives `docker compose down` but a container's
writable layer doesn't.** A named volume's actual data lives in a
directory on the host (`/var/lib/docker/volumes/<name>/_data`) that
exists completely independently of any container's lifecycle — it's
created once by `docker volume create` (or implicitly by Compose) and
only ever removed by an explicit `docker volume rm` (or `docker compose
down -v`, which does that on your behalf). A container's writable layer,
in contrast, is an overlayfs "upper directory" tied one-to-one to that
specific container's identity, torn down the moment `docker rm` deletes
it. Mounting a named volume at `/var/lib/postgresql/data` bypasses that
container-scoped upper directory entirely for that path — the volume's
host directory is bind-mounted directly into the container's mount
namespace at that path, so writes there go straight to storage that isn't
part of the overlayfs layer stack at all and isn't affected by
recreating, or even completely removing, the container that mounts it.

## Exercise

Build a three-service Compose file (`proxy`, `app`, `db`) where `proxy`
and `app` share a `public` network, `app` and `db` share an isolated
`internal: true` network named `data`, and `proxy` has no route to `db`
at all. Verify with `docker compose exec proxy sh -c "getent hosts db"`
failing to resolve, while the same command from `app` succeeds.
