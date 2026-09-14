# 09 · Secrets & Config Management

Plain environment variables (module 03 of Level 2) are visible via
`docker inspect` to anyone with Engine API access, and get written into
Swarm's Raft-replicated state and into process listings on some
platforms. **Docker secrets** and **configs** are purpose-built for
sensitive and non-sensitive external data respectively, mounted as files
rather than passed as environment variables.

## Docker secrets (Swarm mode)

```bash
echo "supersecretpassword" | docker secret create db_password -
docker secret ls
```

```bash
docker service create --name db \
  --secret db_password \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  postgres:16
```

The secret is mounted at `/run/secrets/db_password` inside the
container — an in-memory (`tmpfs`) file, never written to the
container's own writable layer on disk. `POSTGRES_PASSWORD_FILE` is a
convention many official images support specifically for this pattern:
read the actual secret value from a file path given by an env var,
rather than putting the secret value itself in an env var.

## Configs: the non-sensitive counterpart

```bash
docker config create nginx_conf ./nginx.conf
docker service create --name web \
  --config source=nginx_conf,target=/etc/nginx/nginx.conf \
  nginx:1.25
```

A **config** works identically to a secret mechanically (both are
managed, versioned, mounted-as-a-file Swarm objects) but isn't
encrypted at rest with the same guarantees, and is intended for
non-sensitive configuration files (an nginx config, a feature-flag
JSON) rather than credentials.

## Secrets and configs with `docker stack deploy`

```yaml
# stack.yml
services:
  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
    secrets:
      - db_password

  web:
    image: nginx:1.25
    configs:
      - source: nginx_conf
        target: /etc/nginx/nginx.conf

secrets:
  db_password:
    external: true

configs:
  nginx_conf:
    file: ./nginx.conf
```

`external: true` on the secret means it must already exist (created via
`docker secret create` beforehand) — Compose/Swarm deliberately doesn't
let you define a raw secret *value* inline in a stack file, since that
file is exactly the kind of thing that ends up committed to version
control.

## Plain Compose (non-Swarm) secrets support

Compose v2 also supports a lighter-weight `secrets:` block for a single
host, backed by files rather than Swarm's managed secret store:

```yaml
services:
  api:
    build: .
    secrets:
      - api_key

secrets:
  api_key:
    file: ./secrets/api_key.txt
```

This still mounts the secret at `/run/secrets/api_key` inside the
container, but the "management" here is just Compose bind-mounting a
local file — appropriate for local development, not a substitute for
Swarm's encrypted, replicated secret store in a real multi-node cluster.

## Worked example: rotating a secret without downtime

```bash
echo "newpassword" | docker secret create db_password_v2 -

docker service update \
  --secret-rm db_password \
  --secret-add source=db_password_v2,target=db_password \
  db
```

`target=db_password` keeps the in-container file path identical
(`/run/secrets/db_password`) even though the underlying secret object's
name changed — the application never needs to know the secret was
rotated, only that the file at the same path now contains new content,
and Swarm performs this as a rolling update of the service's tasks.

## How It Actually Works

**Why secrets are transmitted and stored encrypted, specifically, not
just access-controlled.** In Swarm mode, the manager's Raft log (module
08) is itself encrypted at rest using a cluster-wide encryption key, and
secret values are additionally encrypted separately using per-secret
keys derived from that same root key — meaning a secret's value never
exists in the Raft log's plaintext even to someone with raw filesystem
access to a manager's data directory. When a task needing a secret is
scheduled onto a worker, the manager decrypts it and transmits it to
that specific worker only over the already-mutually-TLS-authenticated
manager-worker control channel, where the worker's Engine writes it into
a `tmpfs` mount for that one container — never touching that worker's
own persistent disk at any point in the journey. This is meaningfully
different from an environment variable, which travels in whatever
representation the caller used to set it and lands in `docker inspect`
output and the container's own `/proc/<pid>/environ` in cleartext.

**Why `tmpfs` (not a regular bind mount) is what backs `/run/secrets`.**
`tmpfs` is a Linux filesystem type backed entirely by RAM (and swap, if
needed) rather than by any block device — files written there are never
persisted to disk by the kernel at all, and vanish the instant the
`tmpfs` instance is unmounted (i.e., when the container using it is
removed). Mounting the secret this way means even a full disk-image
forensic copy of the host, taken while the container is stopped, would
find no trace of the secret's contents — the strongest guarantee
available short of never materializing the plaintext value on that host
at all.

## Exercise

Create a Docker secret for an API key, deploy a single-replica Swarm
service that mounts it, `docker exec` in to confirm it appears at
`/run/secrets/<name>` with the expected content, then use
`docker service update --secret-rm/--secret-add` to rotate it to a new
value under the same in-container path, and confirm — without
restarting your `exec` session, since the file path is what stays
constant — that a fresh read of that path after the update shows the
new value.
