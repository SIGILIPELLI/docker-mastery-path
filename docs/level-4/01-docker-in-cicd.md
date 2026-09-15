---
description: "Docker in CI/CD Pipelines — Building, testing, and pushing images should happen automatically on every commit, not by hand from a developer's laptop. This…"
---

# 01 · Docker in CI/CD Pipelines

Building, testing, and pushing images should happen automatically on
every commit, not by hand from a developer's laptop. This module covers
the pattern shared by nearly every CI system, illustrated with GitHub
Actions.

## The core pipeline shape

```yaml
# .github/workflows/build.yml
name: Build and Push
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to registry
        uses: docker/login-action@v3
        with:
          registry: registry.example.com
          username: ${{ secrets.REGISTRY_USER }}
          password: ${{ secrets.REGISTRY_TOKEN }}

      - name: Build and test
        run: |
          docker build -t myapp:${{ github.sha }} --target test .
          docker run --rm myapp:${{ github.sha }}-test

      - name: Build production image
        run: docker build -t registry.example.com/team/myapp:${{ github.sha }} .

      - name: Push
        run: docker push registry.example.com/team/myapp:${{ github.sha }}
```

Building against the **exact commit SHA** as the tag (module 08 of Level
2's tagging strategy) means every pushed image is traceable back to the
exact source that produced it — essential once something needs
debugging in production weeks later.

## Running tests inside the build, via a dedicated stage

```dockerfile
FROM node:20 AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM deps AS test
COPY . .
RUN npm test

FROM deps AS build
COPY . .
RUN npm run build

FROM node:20-slim
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./
RUN npm ci --omit=dev
CMD ["node", "dist/server.js"]
```

```bash
docker build --target test -t myapp:test .
docker run --rm myapp:test
```

A `test` stage that never becomes part of the shipped image (module 01
of Level 2's multi-stage pattern) means CI can fail the build on test
failure *before* ever producing the artifact that would otherwise get
pushed — tests run against the exact same filesystem/dependency state
the production image is built from, not a separately maintained CI
environment that could drift from it.

## Using the build cache across CI runs

```yaml
      - name: Build with layer cache
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: registry.example.com/team/myapp:${{ github.sha }}
          cache-from: type=registry,ref=registry.example.com/team/myapp:buildcache
          cache-to: type=registry,ref=registry.example.com/team/myapp:buildcache,mode=max
```

CI runners are typically ephemeral — no local build cache survives
between runs by default — so `cache-from`/`cache-to` explicitly persist
the build cache to the registry itself, letting the next run's `RUN npm
ci` layer (module 04 of Level 2) hit cache even on a brand-new runner
instance.

## Worked example: promote-by-retag across environments in CI

```yaml
  deploy-staging:
    needs: build
    steps:
      - run: |
          docker pull registry.example.com/team/myapp:${{ github.sha }}
          docker tag registry.example.com/team/myapp:${{ github.sha }} registry.example.com/team/myapp:staging
          docker push registry.example.com/team/myapp:staging
```

This is the exact promote-by-retag pattern from Level 2, automated:
staging and (eventually) production both point at the identical,
already-built and already-tested image digest rather than triggering
separate builds per environment.

## How It Actually Works

**Why a registry-backed cache, specifically, is required for ephemeral
CI runners.** BuildKit's cache-from/cache-to normally reads/writes the
local `docker buildx` cache store on disk (module 04 of Level 2's
`--mount=type=cache` covers a different, complementary cache). A fresh
CI runner has no such local store — it's a new filesystem every run. The
`type=registry` cache exporter instead serializes cache layer metadata
and blobs as an OCI image pushed to a registry under a dedicated tag
(`:buildcache` here); the next run's `cache-from` pulls that same
manifest first and primes BuildKit's local cache from it before starting
the actual build, making the registry act as a persistence layer for
otherwise-ephemeral build state. `mode=max` additionally caches
intermediate layers from multi-stage builds that wouldn't otherwise be
exported (by default only the final stage's layers are cache-exported),
at the cost of a larger cache image.

**Why testing inside a dedicated Docker stage, not the CI runner's own
environment, closes a real class of bugs.** A CI runner's own installed
language runtime/library versions can silently drift from what's
actually declared in the Dockerfile (a different Node minor version, a
missing system library the image installs explicitly) — tests passing on
the runner's ambient environment but failing in the shipped container is
a common, painful class of "works in CI, breaks in prod" bug. Running
`npm test` as a Dockerfile stage means the test execution environment is
byte-for-byte the same base image, same `RUN` commands, same file layout
as the production stage that inherits from the identical earlier layers
— any environment-caused failure surfaces during the build itself, in
CI, rather than after deployment.

## Exercise

Add a `test` stage to a multi-stage Dockerfile for any small app, wire a
GitHub Actions (or equivalent) workflow that runs `docker build --target
test` and fails the job on a nonzero exit code, and verify by
deliberately breaking a test that the workflow correctly stops before
reaching the push step.
