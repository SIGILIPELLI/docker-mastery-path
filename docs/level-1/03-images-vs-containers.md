# 03 · Images vs Containers

This distinction trips up almost everyone at first, so it's worth making
precise before writing a single Dockerfile.

## The core idea

An **image** is a read-only template: a stack of filesystem layers plus
metadata (default command, exposed ports, environment variables, etc.)
that describes how to create a container. It is inert — it doesn't run,
consume CPU, or hold state. Think of it like a class definition, or like a
frozen snapshot of a filesystem plus instructions for how to start a
process from it.

A **container** is a running (or stopped) *instance* created from an
image — analogous to an object instantiated from a class. Many containers
can be created from the same image, each with its own writable layer, its
own process, its own network address, and its own lifecycle, without
affecting the image or each other.

```
   image (read-only)              containers (instances)
  ┌──────────────────┐         ┌─────────────────────────┐
  │ layer: base OS    │         │ container A             │
  │ layer: deps       │  --->   │  image layers (shared,  │
  │ layer: app code   │  --->   │  read-only) + its own   │
  │ metadata: CMD ...  │         │  thin writable layer    │
  └──────────────────┘         ├─────────────────────────┤
                                │ container B             │
                                │  same shared layers +    │
                                │  a *different* writable  │
                                │  layer                   │
                                └─────────────────────────┘
```

## Layers, concretely

Each instruction in a Dockerfile that changes the filesystem (`RUN`,
`COPY`, `ADD`) produces a new, immutable layer, identified by a content
hash. Layers are stacked with a union filesystem so that from inside the
container it looks like one normal filesystem, but on disk Docker only
stores each distinct layer once — even across different images, if they
happen to share layers (e.g. two images both built `FROM python:3.12-slim`
share that base layer on disk).

When you start a container, Docker adds one more layer on top: a thin,
writable layer unique to that container. Any file the container creates,
modifies, or deletes at runtime happens in this writable layer — the
underlying image layers are never touched. This is why:

- Multiple containers from the same image don't interfere with each
  other's runtime changes.
- Deleting a container discards its writable layer (and any data written
  only there) but leaves the image untouched.
- Two images sharing base layers take much less disk space combined than
  their sizes would suggest if added naively.

## Seeing this yourself

```bash
# List images you have locally
docker images
# REPOSITORY   TAG       IMAGE ID       CREATED        SIZE
# python       3.12-slim 6f2a...        2 weeks ago    130MB

# Create (but don't start) a container from an image
docker create --name demo python:3.12-slim
# prints a long container ID

# List all containers, including stopped ones
docker ps -a
# CONTAINER ID   IMAGE               STATUS    NAMES
# 3f9c1a2b...    python:3.12-slim    Created   demo

# Start it, run something, and it writes to its own layer
docker start -ai demo
```

`docker images` and `docker ps -a` are the two commands to keep straight:
`images` lists templates; `ps -a` lists instances (running or not). The
`-a` flag on `ps` matters — without it, Docker only shows *running*
containers, hiding stopped ones.

## One image, many containers

```bash
docker run -d --name web1 nginx
docker run -d --name web2 nginx
docker ps
# CONTAINER ID   IMAGE   STATUS    NAMES
# a1b2c3...      nginx   Up ...    web1
# d4e5f6...      nginx   Up ...    web2
```

Both `web1` and `web2` were created from the single locally cached `nginx`
image. Stopping and removing `web1` has no effect on `web2` or on the
`nginx` image itself — `docker images` will still show `nginx` present
after both containers are removed.

## Image identity: repository, tag, and digest

An image reference like `python:3.12-slim` has two parts:

- **Repository** (`python`) — the image's name.
- **Tag** (`3.12-slim`) — a human-friendly version label. If omitted,
  Docker assumes `:latest`, which is just a conventional tag name, not a
  magic "always newest" pointer — it's whatever image was last pushed
  with that tag.

Every image also has an immutable **digest** (a SHA-256 hash of its
manifest), which is the only fully unambiguous way to refer to a specific
image content — two different tags can point at the very same digest, and
a tag can be *re-pushed* to point at a new digest later, which tags alone
can't protect you against.

```bash
docker images --digests python
# REPOSITORY   TAG         DIGEST                                          IMAGE ID
# python       3.12-slim   sha256:1a2b3c...                                6f2a...
```

## Cheat sheet

| Command | Operates on |
|---|---|
| `docker images` / `docker image ls` | Images |
| `docker rmi <image>` | Images |
| `docker pull` / `docker push` | Images |
| `docker ps -a` / `docker container ls -a` | Containers |
| `docker run`, `docker start`, `docker stop` | Containers |
| `docker rm <container>` | Containers |

## Exercise

Run three containers from the same `nginx` image, give each a distinct
name, list them with `docker ps`, then remove just one with
`docker rm -f <name>`. Confirm with `docker images` that the `nginx`
image is still present, and with `docker ps -a` that the other two
containers are unaffected. This should make the image/container split
concrete: one template, several independent running instances, and
removing an instance never touches the template.
