---
description: "Disaster Recovery & Backups — Everything containerized is, by design, disposable — a container can be destroyed and recreated from its image in seconds…"
---

# 09 · Disaster Recovery & Backups

Everything containerized is, by design, disposable — a container can be
destroyed and recreated from its image in seconds (that's the whole point
of module 07's zero-downtime replacement). But two things are *not*
disposable that way: **data in volumes** (databases, uploaded files) and
**images in a private registry** (module 05) that aren't reproducible from
source alone if the build context or base image ever disappears. This
module covers backing up both, and actually restoring from the backup —
a backup that's never been restored is a hope, not a plan.

## What actually needs backing up

- **Named volumes** — anything a database or stateful service writes to
  disk (module 06's storage drivers make clear this data lives outside
  the container's writable layer, on the host or a plugin-backed store).
- **Registry contents** (module 05) — image tags, especially ones built
  from a Dockerfile whose exact base-image digest or build context may no
  longer be reconstructible later (a base image tag can be repointed
  upstream; a digest-pinned build from six months ago may not be).
- **Compose/stack files and `.env` secrets material** — the deployment
  definition itself, usually already in git, but explicitly verify it is.

Container images and containers themselves are *not* what you back up —
they're rebuilt from source and redeployed from the stack file.

## Backing up a named volume

```bash
docker run --rm \
  -v pgdata:/data:ro \
  -v "$(pwd)/backups:/backup" \
  alpine tar czf /backup/pgdata-$(date +%Y%m%d).tar.gz -C /data .
```

This is the standard "sidecar container" backup pattern: a throwaway
`alpine` container mounts the volume you want to back up (read-only, so
the backup process itself can't corrupt live data) alongside a host
directory, and `tar` streams the volume's contents into a timestamped
archive on the host. No changes to the running database container are
needed — the volume is a first-class Docker object independent of which
container currently has it mounted.

```bash
ls backups/
# pgdata-20260910.tar.gz  pgdata-20260909.tar.gz  pgdata-20260908.tar.gz
```

## Restoring a volume from backup

```bash
docker volume create pgdata_restored
docker run --rm \
  -v pgdata_restored:/data \
  -v "$(pwd)/backups:/backup" \
  alpine tar xzf /backup/pgdata-20260910.tar.gz -C /data
```

Restoring into a *new* volume name first (rather than overwriting the
live one in place) lets you verify the restore succeeded before cutting
anything over — point a throwaway container at `pgdata_restored` and
confirm the data looks right before ever touching production.

## Database-aware backups (when file-level isn't consistent enough)

A running database's on-disk files can be mid-write when a `tar` backup
runs, producing a backup that looks complete but is internally
inconsistent. Prefer the database's own consistent-snapshot tooling when
one exists:

```bash
docker exec postgres-db pg_dump -U postgres mydb | gzip > backups/mydb-$(date +%Y%m%d).sql.gz

# restore
gunzip -c backups/mydb-20260910.sql.gz | docker exec -i postgres-db psql -U postgres mydb
```

`pg_dump` reads through Postgres's own transaction-consistent view of the
data rather than the raw files on disk, so the resulting dump is
guaranteed internally consistent regardless of what writes were in flight
when it started — something the raw `tar` approach can't promise for a
live, actively-written database.

## Backing up a private registry (module 05)

```bash
docker run -d -p 5000:5000 --name registry \
  -v registry-data:/var/lib/registry \
  registry:2

# back up the registry's own storage the same way as any named volume
docker run --rm -v registry-data:/data:ro -v "$(pwd)/backups:/backup" \
  alpine tar czf /backup/registry-$(date +%Y%m%d).tar.gz -C /data .
```

For a cloud-hosted registry rather than a self-run one, the equivalent is
scripted re-tagging and pushing of every tag you depend on to a second
registry (or `docker save`/`docker load` to portable tarballs) — the goal
is the same either way: don't let "the only copy of this image" live in
one place with one provider.

## Worked example: full disaster recovery drill

```bash
# 1. Simulate the disaster: the host is gone
docker compose down -v   # -v also removes the named volumes -- total loss

# 2. Recreate the stack from source of truth (git, not memory)
git clone https://example.com/myapp.git && cd myapp

# 3. Recreate volumes and restore data into them before starting services
docker volume create pgdata
docker run --rm -v pgdata:/data -v "$(pwd)/backups:/backup" \
  alpine tar xzf /backup/pgdata-20260910.tar.gz -C /data

# 4. Bring the stack up against the restored volume
docker compose up -d

# 5. Verify -- don't just assume the restore worked
docker compose exec db psql -U postgres -c "SELECT count(*) FROM users;"
# compare the count against what you expect from the backup's timestamp
```

Step 5 is the step most real incidents skip under pressure, and it's the
one that actually determines whether the recovery worked.

## How It Actually Works

**Why a volume backup is safe to take with `-v pgdata:/data:ro` while the
database container keeps running.** A named volume (module 06) is a
directory managed by Docker's volume driver, bind-mounted by the kernel
into as many containers as reference it simultaneously — mounting it
read-only into the backup sidecar doesn't pause or lock it for the
database container, which still has its own (typically read-write) mount
of the same underlying directory. The risk isn't a mount conflict, it's a
*point-in-time consistency* one: `tar` reads files sequentially and a
database can modify files between when `tar` reads file A and when it
reads file B, producing an archive where A and B were never simultaneously
true on disk — which is exactly why the module recommends `pg_dump`-style
logical backups for anything actively written, since those go through the
database engine's own MVCC snapshot rather than racing its file writes at
the filesystem level.

**Why restoring into a fresh volume name, not overwriting live data, is
the mechanically safer default.** `docker volume create` allocates a new,
empty directory under the volume driver's storage root, entirely separate
from any existing volume — there is no shared state between
`pgdata` and `pgdata_restored` until you explicitly point a container at
one or the other. This means a bad or partial restore is contained to the
new volume and never touches the original, and cutover becomes a single
atomic decision (which volume name does the container mount) rather than
an in-place overwrite that can't be undone if the restored data turns out
to be wrong.

## 🔀 Related lessons on other tracks

- [AWS — Multi-Region & Disaster Recovery](https://sigilipelli.github.io/aws-mastery-path/level-3/07-multi-region-disaster-recovery/)
- [Azure — 07 · High Availability & Disaster Recovery](https://sigilipelli.github.io/azure-mastery-path/level-3/07-ha-disaster-recovery/)
- [GCP — 07 · Multi-Region & Disaster Recovery](https://sigilipelli.github.io/gcp-mastery-path/level-3/07-multi-region-disaster-recovery/)

## Exercise

Stand up a Postgres container with a named volume, insert a few rows,
take both a `tar`-based volume backup and a `pg_dump` logical backup, then
run `docker compose down -v` to simulate total loss. Restore from the
`pg_dump` backup into a fresh container and volume, and confirm via `SELECT
count(*)` that the row count matches what you inserted before the
simulated disaster.
