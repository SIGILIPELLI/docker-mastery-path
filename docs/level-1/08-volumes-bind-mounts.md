# 08 · Volumes & Bind Mounts

By default, anything a container writes lives in its writable layer, and
is destroyed along with the container when it's removed (module 03). For
data that needs to outlive a container — a database's files, uploaded
content — Docker offers two main mechanisms for durable, external storage:
**volumes** and **bind mounts**.

## The problem they solve

```bash
docker run -d --name db1 postgres:16
# ... write some data into the database ...
docker rm -f db1
docker run -d --name db2 postgres:16
# db2 starts with a totally fresh, empty database -- db1's data is gone
```

Without external storage, a container's data has the same lifetime as the
container itself. Volumes and bind mounts detach the *data's* lifetime
from the *container's* lifetime.

## Volumes

A **volume** is storage managed entirely by Docker, stored in a location
under Docker's control on the host (e.g.
`/var/lib/docker/volumes/...` on Linux), and referenced by name rather
than by host path.

```bash
docker volume create pgdata
docker volume ls
docker volume inspect pgdata

docker run -d --name db1 -v pgdata:/var/lib/postgresql/data postgres:16
```

`-v pgdata:/var/lib/postgresql/data` mounts the volume named `pgdata` at
`/var/lib/postgresql/data` inside the container — the path Postgres
stores its data files under. Now:

```bash
docker rm -f db1
docker run -d --name db2 -v pgdata:/var/lib/postgresql/data postgres:16
```

`db2` starts up and finds `db1`'s data already there, because both
containers were pointed at the same named volume, and the volume's
lifetime is independent of either container.

If you use `-v pgdata:/path` and the volume doesn't already exist, Docker
creates it automatically on first use — you don't strictly need
`docker volume create` first, though creating it explicitly makes the
intent clearer.

## Bind mounts

A **bind mount** maps a specific path on the *host* directly into the
container, rather than a Docker-managed volume:

```bash
docker run -d --name web -p 8080:80 \
  -v "$(pwd)/site:/usr/share/nginx/html:ro" \
  nginx
```

This mounts the host directory `./site` at `/usr/share/nginx/html` inside
the container, read-only (`:ro`). Any file you edit in `./site` on the
host is immediately visible inside the running container, and vice versa
for writes (when not read-only) — because it's literally the same
underlying filesystem location, just visible from two mount points.

## Volumes vs bind mounts

| | Volume | Bind mount |
|---|---|---|
| Managed by | Docker | You (arbitrary host path) |
| Location | Docker-controlled directory | Any host path you specify |
| Portability | Portable across hosts running Docker (path is abstracted) | Tied to that host's specific filesystem layout |
| Typical use | Databases, persistent application state | Local development (live-editing source code), injecting host config files |
| Referenced as | `-v <volume-name>:<container-path>` | `-v <host-absolute-path>:<container-path>` |

A quick rule of thumb: reach for a **volume** when you want Docker to
manage durable data for you (production databases, caches); reach for a
**bind mount** when you specifically need to expose a known host path
(mounting your live source tree during development, or feeding in a
config file from a known location).

## The `--mount` syntax (more explicit alternative)

```bash
docker run -d --name db1 \
  --mount type=volume,source=pgdata,target=/var/lib/postgresql/data \
  postgres:16

docker run -d --name web \
  --mount type=bind,source="$(pwd)/site",target=/usr/share/nginx/html,readonly \
  -p 8080:80 nginx
```

`--mount` is more verbose but unambiguous about `type=volume` vs
`type=bind` — Docker's documentation recommends it for clarity, especially
in scripts, though the shorter `-v` form remains extremely common and
both are fully supported.

## Inspecting and cleaning up

```bash
docker volume ls
docker volume inspect pgdata
docker inspect -f '{{json .Mounts}}' db1   # shows all mounts on a container

docker volume rm pgdata          # fails if a container is still using it
docker volume prune              # removes all volumes not used by any container
```

Removing a container does **not** remove its named volumes by default —
this is intentional, since the whole point of a volume is to outlive the
container. To remove a container *and* its anonymous volumes together,
use `docker rm -v` (only affects volumes that were created anonymously
for that container, not named ones you created separately).

## Worked example: persistent counter

```bash
docker volume create counter-data

docker run --rm -v counter-data:/data alpine sh -c \
  'n=$(cat /data/count 2>/dev/null || echo 0); echo $((n+1)) > /data/count; cat /data/count'
# 1

docker run --rm -v counter-data:/data alpine sh -c \
  'n=$(cat /data/count 2>/dev/null || echo 0); echo $((n+1)) > /data/count; cat /data/count'
# 2
```

Each `docker run --rm` here creates and destroys a *container*, but
because both mount the same `counter-data` volume, the count persists and
increments across completely separate containers — proof the data's
lifetime is decoupled from any single container's lifetime.

## How It Actually Works

Volumes and bind mounts both work by exploiting the **mount namespace**
(`CLONE_NEWNS`) the same way the container's root filesystem does — but
they bypass overlayfs entirely for the paths involved, which is precisely
why they behave differently from regular container writes.

**A bind mount is a second view of the same inode, not a copy.** The
Linux `mount --bind` mechanism (which `docker run -v host:container`
ultimately invokes inside the container's mount namespace) doesn't copy
data anywhere — it makes an existing directory (or file) accessible at a
second path by pointing a new mount point at the same underlying inode on
the host's real filesystem. Reads and writes from either path hit the
identical on-disk blocks. That's why edits made from the host appear
instantly inside the container and vice versa: there's exactly one copy
of the data, referenced from two locations, with no overlayfs upper/lower
layering involved for that specific mounted path at all — it's excluded
from the union entirely and mounted directly over whatever was there.

**A named volume is the same mechanism, pointed at Docker's own
directory.** `docker volume create pgdata` just makes a directory under
`/var/lib/docker/volumes/pgdata/_data` on the host. `-v pgdata:/var/lib/postgresql/data`
performs the identical bind-mount operation as above, with the volume's
managed directory as the host side — the only difference from a bind
mount is *who* picked the host path (Docker, vs. you). This is also why
volume data survives `docker rm`: removing a container tears down its
mount namespace (the mount points disappear), but the underlying
directory on the host — the actual inode holding the data — is untouched,
exactly as unmounting a USB drive doesn't erase the drive.

**Why the data survives even though the image layers don't.** A
container's writable layer is an overlayfs `upperdir` that gets deleted
along with the container. A mounted path is deliberately carved out of
that overlay — the kernel resolves any path under a mount point by
following the mount table first, so the overlayfs layers underneath a
bind-mounted directory are never consulted or written to for that path at
all while the mount is active.

## Exercise

Create a named volume called `notes-data`. Run an `alpine` container that
mounts it at `/data` and appends a line of text to `/data/notes.txt`
(e.g. via `echo "first run" >> /data/notes.txt`), then exits with `--rm`.
Run the same command two more times with different text. Finally, run one
more `--rm` container mounting the same volume and `cat /data/notes.txt`
to confirm all three lines are present, proving the volume persisted data
across three separate, fully-removed containers.
