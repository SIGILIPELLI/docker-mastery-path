---
description: "Build Optimization at Scale — A single team's Dockerfile might build in seconds. A large codebase with many services, many CI jobs running concurrently…"
---

# 04 · Build Optimization at Scale

A single team's Dockerfile might build in seconds. A large codebase with
many services, many CI jobs running concurrently, and a large monorepo
needs its build infrastructure treated as a system to optimize, not just
individual Dockerfiles.

## Shared build caches across many CI runners

Module 01's `cache-from`/`cache-to` with `type=registry` scales to many
concurrent runners sharing one cache image, but a single cache tag
becomes a write-contention bottleneck under heavy concurrency. A common
refinement is per-branch cache tags with a fallback chain:

```yaml
      - uses: docker/build-push-action@v5
        with:
          cache-from: |
            type=registry,ref=registry.example.com/team/myapp:buildcache-${{ github.ref_name }}
            type=registry,ref=registry.example.com/team/myapp:buildcache-main
          cache-to: type=registry,ref=registry.example.com/team/myapp:buildcache-${{ github.ref_name }},mode=max
```

Each branch reads its own cache first, falling back to `main`'s cache for
layers it hasn't built yet (e.g. a new branch's first build) — this
keeps feature-branch builds fast without every branch fighting over one
shared, frequently-overwritten cache tag.

## Remote builders for consistent, powerful build hosts

```bash
docker buildx create --name remote-builder \
  --driver remote tcp://build-farm.internal:1234
docker buildx use remote-builder
docker buildx build --push -t registry.example.com/team/myapp:1.0 .
```

A **remote builder** runs BuildKit on dedicated, consistently-provisioned
hardware rather than whatever CI runner happened to be assigned —
valuable when CI runners are small/ephemeral but builds need substantial
CPU/memory (large monorepo builds, heavy compilation), or when you want
one warm, persistent build cache shared across every CI job rather than
re-establishing it per runner via the registry each time.

## Parallelizing builds across a monorepo

```bash
# Build only the services whose source actually changed
CHANGED=$(git diff --name-only origin/main... | cut -d/ -f1 | sort -u)
for svc in $CHANGED; do
  docker buildx build --push -t registry.example.com/team/$svc:${GITHUB_SHA} ./$svc &
done
wait
```

Building every service on every commit regardless of what changed wastes
CI time linearly with monorepo size. Detecting the changed subset (via
`git diff` against the merge-base, as above, or a purpose-built tool like
Bazel/Turborepo/Nx for dependency-aware change detection) and building
only those, in parallel, keeps CI time roughly proportional to the size
of a single change rather than the whole repo.

## Multi-platform builds without duplicating CI runners

```bash
docker buildx build --platform linux/amd64,linux/arm64 \
  --push -t registry.example.com/team/myapp:1.0 .
```

Buildx with the `docker-container` driver can build multiple target
architectures from one invocation, using QEMU emulation (or native
builders per architecture registered as separate nodes for better
performance) and pushing a single **manifest list** that lets `docker
pull` on either architecture automatically fetch the right variant —
avoiding a separate CI job/runner type per target architecture for
straightforward cases.

## Worked example: measuring the effect of a warm shared cache

```bash
# Cold cache (first build on a fresh runner)
time docker buildx build --cache-from type=registry,ref=...:buildcache .
# e.g. 4m12s for a large dependency-heavy image

# Warm cache (subsequent build, no dependency changes)
time docker buildx build --cache-from type=registry,ref=...:buildcache .
# e.g. 18s — nearly every layer hits cache
```

(Illustrative figures reasoned through from typical dependency-install
costs, not measured on a specific machine — the ratio, not the absolute
numbers, is the point: a warm cache typically turns minutes into seconds
for unchanged layers.)

## How It Actually Works

**Why a `type=registry` cache export writes a real, separate image
manifest rather than reusing the application image's own layers
directly.** BuildKit's registry cache exporter serializes its internal
cache metadata (a graph of which build steps produced which layer
digests, keyed by the exact instruction+input hash from Level 2 module
04) as a specially-structured OCI image, pushed under the `:buildcache`
tag, distinct from the application's own tagged images. On the next
build, `cache-from` pulls just that manifest and metadata (not
necessarily every blob immediately — BuildKit lazily fetches only the
layer blobs it actually needs to satisfy a cache hit) and reconstructs
its local cache index from it before evaluating which steps can be
skipped. This separation is why the cache image can grow to include far
more historical layers than any single application image would ever
need (with `mode=max` capturing every intermediate stage's layers, per
module 01) without bloating the actual deployed image — they are
entirely separate registry objects.

**Why QEMU-based cross-architecture builds are slower, mechanically, than
native per-architecture builders.** When building `linux/arm64` on an
`amd64` CI runner, `RUN` instructions for that stage execute inside a
container whose binaries are ARM64 machine code — the `amd64` kernel
cannot execute that code natively, so `binfmt_misc` (a kernel mechanism
mapping an executable's magic bytes to a registered interpreter) invokes
QEMU's user-mode emulator to translate each ARM64 instruction to `amd64`
instructions dynamically, instruction-by-instruction, as the process
runs. This per-instruction translation overhead is why compilation-heavy
`RUN` steps under emulation can run several times slower than the
equivalent native build — registering a genuinely separate `arm64` build
node (real ARM64 hardware or a cloud ARM64 instance) as an additional
buildx builder avoids emulation entirely for that platform's stages,
trading additional infrastructure for native-speed builds.

## Exercise

Set up a `docker buildx build --platform linux/amd64,linux/arm64` for any
small multi-stage Dockerfile, time the `RUN npm ci`-equivalent step under
QEMU emulation for the non-native architecture, then (if you have access
to real ARM64 hardware, or an ARM64 cloud instance) register it as a
second builder node and compare the same step's wall-clock time running
natively.
