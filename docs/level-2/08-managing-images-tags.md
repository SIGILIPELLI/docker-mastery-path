---
description: "Managing Images & Tags — An image name like myapp:1.4.2 has two parts — the repository (myapp) and the tag (1.4.2) — and how you assign and clean up tags…"
---

# 08 · Managing Images & Tags

An image name like `myapp:1.4.2` has two parts — the **repository**
(`myapp`) and the **tag** (`1.4.2`) — and how you assign and clean up tags
directly affects whether you can reliably reproduce what's running in
production, and how much disk space your build hosts accumulate.

## Tagging strategies

```bash
docker build -t myapp:1.4.2 .
docker tag myapp:1.4.2 myapp:latest
docker tag myapp:1.4.2 registry.example.com/team/myapp:1.4.2
```

`docker tag` doesn't copy anything — it adds another name pointing at the
same image ID, the way a hard link adds another filename for the same
inode.

| Strategy | Example | Trade-off |
|---|---|---|
| Semantic version | `myapp:1.4.2` | Precise, reproducible, but you must remember to bump it |
| Git SHA | `myapp:a1b2c3d` | Always unique, traceable to exact source, unreadable to humans |
| `latest` | `myapp:latest` | Convenient default, but **mutable** — it silently points at whatever was last pushed, which is exactly why it should never be the tag a production deployment pins to |
| Environment | `myapp:staging`, `myapp:prod` | Readable, but also mutable — same caveat as `latest` |

A robust pattern combines them: build and push both `myapp:a1b2c3d` (immutable, for deploys to pin to) and `myapp:1.4.2` (human-readable) pointing at the same image, and only move `myapp:latest` to point at the newest release for convenience during local development.

## Listing and cleaning up images

```bash
docker images                        # every locally stored image
docker images myapp                  # filter by repository
docker images --filter "dangling=true"   # untagged, unreferenced layers
```

A **dangling** image is one with no tag at all (`<none>:<none>`) —
typically left behind when you rebuild `myapp:latest` and the old image
ID that used to hold that tag is now nameless but still on disk.

```bash
docker image prune                   # remove all dangling images
docker image prune -a                # remove ALL images not used by a running container
docker rmi myapp:1.3.0               # remove one specific tag
```

`docker image prune -a` is far more aggressive than plain `docker image
prune` — it also removes tagged images that simply aren't currently
backing a running container, which is appropriate for a CI build agent
that should stay clean between jobs but dangerous on a workstation where
you keep images around between sessions.

## Pushing and pulling by tag

```bash
docker tag myapp:1.4.2 registry.example.com/team/myapp:1.4.2
docker push registry.example.com/team/myapp:1.4.2
docker pull registry.example.com/team/myapp:1.4.2
```

## Worked example: promoting an image across environments

```bash
# Build once
docker build -t registry.example.com/team/myapp:a1b2c3d .
docker push registry.example.com/team/myapp:a1b2c3d

# "Promote" to staging by tagging the SAME image, not rebuilding it
docker pull registry.example.com/team/myapp:a1b2c3d
docker tag registry.example.com/team/myapp:a1b2c3d registry.example.com/team/myapp:staging
docker push registry.example.com/team/myapp:staging

# Later, promote the identical, already-tested artifact to prod
docker tag registry.example.com/team/myapp:a1b2c3d registry.example.com/team/myapp:prod
docker push registry.example.com/team/myapp:prod
```

Retagging the exact same image ID for each environment (rather than
rebuilding per environment) is what guarantees "what we tested in staging
is byte-for-byte what's running in production."

## How It Actually Works

**A tag is a pointer, and pointers can be moved without touching any
content.** Internally, the daemon (and the registry) key everything by
**digest** — a SHA-256 hash of the image manifest, which itself lists the
content-addressed hashes of every layer. A tag like `myapp:1.4.2` is
stored as a mutable mapping from that human-readable string to a specific
digest, kept in the daemon's local image store (and, for a registry, in
its own tag-to-manifest index). `docker tag` writes a new entry in that
mapping pointing at an existing digest — no bytes are duplicated on disk,
and no network transfer happens. This is exactly why `docker pull
myapp:latest` twice, an hour apart, can silently hand you two different
sets of bytes despite an identical command: `latest` is just whichever
digest the mapping currently resolves to, and someone could have pushed a
new image under that tag in between — the immutable, verifiable
alternative is to pull by digest directly
(`docker pull myapp@sha256:...`), which can never resolve to different
content.

**Why `docker image prune -a` can free gigabytes instantly.** Layers are
shared, reference-counted content on disk (via the storage driver's
content store, keyed by layer digest) — removing a tag decrements the
reference count for the layers it pointed to, and only layers with a
reference count of zero (not referenced by any remaining tag or by any
container's read-only layer set) are actually deleted from
`/var/lib/docker`. This is why building ten variants of an image that all
`FROM python:3.12-slim` costs almost no extra disk beyond the first —
they all share that base's layers by reference — and why pruning can look
dramatic: a single removed tag can be the last reference holding dozens
of otherwise-orphaned layers, all reclaimed together.

## Exercise

Build the same Dockerfile twice under two different tags without
changing anything (`docker build -t myapp:v1 .` then
`docker build -t myapp:v2 .`), and confirm with `docker images
--digests` that both tags resolve to the identical digest. Then change
one line, rebuild as `myapp:v3`, and use `docker image prune` to observe
that the dangling intermediate layers from the *previous* untagged builds
(if any) get cleaned up, while `v1`/`v2`/`v3` — all currently tagged —
are left alone.
