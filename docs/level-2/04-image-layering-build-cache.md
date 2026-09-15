---
description: "Image Layering & Build Cache — Every instruction in a Dockerfile that changes the filesystem (RUN, COPY, ADD) produces a new, independent…"
---

# 04 · Image Layering & Build Cache

Every instruction in a Dockerfile that changes the filesystem (`RUN`,
`COPY`, `ADD`) produces a new, independent, content-addressed layer.
Docker caches each layer and reuses it on later builds when nothing that
could affect it has changed — which is why instruction *order* has a
direct, measurable effect on build speed.

## The cache invalidation rule

For each instruction, the builder checks whether it can reuse a
previously built layer:

- For `RUN`, the cache key is the exact command string plus the parent
  layer's identity — same command, same starting point, same cached
  result reused.
- For `COPY`/`ADD`, the cache key additionally includes a checksum of the
  files being copied — if even one byte of a copied file changed, that
  layer (and every layer after it) is rebuilt.

The critical consequence: **once one layer misses the cache, every
subsequent layer misses too**, even if their own inputs didn't change,
because their "starting point" (the previous layer) is now different.

## Ordering for maximum cache reuse

```dockerfile
FROM python:3.12-slim
WORKDIR /app

# Copy ONLY the dependency manifest first
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Application source changes far more often than dependencies —
# putting COPY . . last means edits to app code never invalidate
# the (usually slow) pip install layer above it.
COPY . .

CMD ["python", "app.py"]
```

Compare to the naive ordering:

```dockerfile
COPY . .                                    # any code change invalidates everything below
RUN pip install --no-cache-dir -r requirements.txt   # reinstalled on every code edit
```

## Worked example: measuring the effect

```bash
# First build: everything is cold
time docker build -t myapp .

# Edit only app.py, leave requirements.txt untouched
docker build -t myapp .
# With the good ordering: pip install layer says "CACHED", build takes ~1s
# With COPY . . first: pip install reruns from scratch, taking as long as build #1
```

## BuildKit cache mounts for package managers

Even with good layer ordering, a *changed* `requirements.txt` still
re-downloads every package from scratch by default. A **cache mount**
persists a directory (like pip's or npm's download cache) across builds,
independent of the layer cache:

```dockerfile
# syntax=docker/dockerfile:1
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN --mount=type=cache,target=/root/.cache/pip \
    pip install -r requirements.txt
```

```dockerfile
FROM node:20
WORKDIR /app
COPY package.json package-lock.json ./
RUN --mount=type=cache,target=/root/.npm \
    npm ci
```

The `--mount=type=cache` directory is *not* committed as part of the
image layer — it's a persistent cache directory reused across separate
builds (even ones that would otherwise be cache misses), separate from
the layer cache that keys off the Dockerfile instruction and input hash.

## How It Actually Works

**Layers are content-addressed, and the cache key chains through
history.** Each layer's ID is a hash of its own diff content combined
with its parent layer's ID — so a layer's identity mechanically encodes
"the entire sequence of instructions that produced it," not just its own
diff. When BuildKit decides whether a `RUN` can reuse cache, it hashes
the instruction text plus that chained parent ID; if the parent ID
differs from last time (because an earlier `COPY` produced different
content), the hash can't match anything in the cache store even though
the `RUN` command's *text* is identical — this is the mechanical reason a
single upstream cache miss cascades forward through the rest of the
Dockerfile.

**Why `--mount=type=cache` survives what the layer cache doesn't.**
BuildKit implements cache mounts as named, persistent directories on the
build host (or a shared cache backend), mounted into the ephemeral build
container for the duration of that one `RUN` instruction only, then
unmounted — so their contents never become part of any layer's diff and
are never hashed into the layer's content-addressed ID. This decouples
"did the layer cache hit" from "is the package-manager download cache
warm": a `requirements.txt` change always invalidates the `RUN pip
install` *layer* (because `COPY requirements.txt .`'s content hash
changed), but the cache-mounted `/root/.cache/pip` directory is untouched
by that invalidation and still has the previously downloaded wheel files
sitting on disk, so `pip` only needs to fetch genuinely new/changed
packages instead of everything.

## Exercise

Take a Dockerfile that does `COPY . .` before `RUN npm install`, and
rewrite it to copy `package.json`/`package-lock.json` first, run
`npm ci` with a `--mount=type=cache,target=/root/.npm`, then copy the
rest of the source. Build twice, editing only a source file (not
`package.json`) between builds, and confirm via the build output that the
`npm ci` layer shows `CACHED` on the second build.
