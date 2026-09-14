# 01 · Multi-Stage Builds

A **multi-stage build** uses more than one `FROM` in a single Dockerfile so
that build tooling (compilers, dev dependencies, source archives) never
ends up in the image you actually ship. Only the artifacts you explicitly
`COPY --from=` between stages survive into the final image.

## The problem it solves

A naive Dockerfile for a compiled language installs the whole toolchain
into the final image:

```dockerfile
FROM golang:1.22
WORKDIR /app
COPY . .
RUN go build -o server .
CMD ["./server"]
```

This works, but `golang:1.22` is over 800MB — the compiler, standard
library sources, and build cache all ship to production even though the
running container only needs one static binary.

## Splitting into stages

```dockerfile
# ---- Stage 1: build ----
FROM golang:1.22 AS builder
WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -o /out/server .

# ---- Stage 2: runtime ----
FROM alpine:3.19
RUN apk add --no-cache ca-certificates
COPY --from=builder /out/server /usr/local/bin/server
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/server"]
```

Each `FROM` starts a fresh, independent build stage with its own
filesystem. `AS builder` names stage 1 so stage 2 can reference it.
`COPY --from=builder /out/server ...` reaches into stage 1's filesystem
and copies out just the compiled binary — none of `golang:1.22`'s 800MB
toolchain crosses into the final image, which now inherits only from the
~7MB `alpine:3.19` base.

## Worked example: a Node.js app with a build step

Frontend and backend frameworks that need a build step (bundling,
transpiling TypeScript) benefit the same way:

```dockerfile
FROM node:20 AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

FROM node:20 AS build
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:20-slim AS runtime
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/package.json ./
RUN npm ci --omit=dev
CMD ["node", "dist/server.js"]
```

The `deps` stage caches `npm ci` separately from source changes, `build`
compiles TypeScript/bundles assets with full `devDependencies` available,
and `runtime` starts clean from a slim base with only production
dependencies and compiled output. (Reasoned through against normal
`npm`/Node conventions; not run for this lesson.)

## Targeting a specific stage

```bash
docker build --target build -t myapp:build-debug .
```

`--target` stops the build at a named stage, which is useful for building
a debug image that keeps devDependencies and source maps without touching
the production Dockerfile stage.

## Copying between stages more than once

You aren't limited to one final stage — you can pull artifacts from
several earlier stages into a common final image:

```dockerfile
FROM golang:1.22 AS backend-builder
...
FROM node:20 AS frontend-builder
...
FROM alpine:3.19
COPY --from=backend-builder /out/server /usr/local/bin/server
COPY --from=frontend-builder /app/dist /usr/share/web
```

## How It Actually Works

**Stages are just independent image builds sharing a build cache.** Each
`FROM` line resets the builder's notion of "current image" to that base,
with its own layer stack. Nothing about a later stage's filesystem is
implicitly visible to it — a `COPY --from=builder` is the *only* channel
between stages, and it works by having BuildKit keep every stage's final
filesystem snapshot addressable (by stage name or index) until the build
finishes, then doing an ordinary layer-diff copy from that snapshot's
merged overlayfs view into the new stage, exactly as if you'd run
`docker cp` from a container built from that stage into the new one. This
is also why an unused stage still gets built (unless BuildKit can prove
nothing references it) — the builder can't know a stage is unreachable
until it has resolved the whole dependency graph of `--from=` references.

**Why the final image genuinely doesn't contain earlier stages' layers.**
An image is nothing more than a manifest listing an ordered set of layer
digests plus config. When stage 2 does `FROM alpine:3.19`, its layer list
starts from alpine's own layers; the `COPY --from=builder` instruction
adds *one new layer* on top, containing only the files that instruction
copied. The builder stage's layers (the Go toolchain, `go mod` cache,
intermediate object files) are never referenced by the final manifest, so
they are not pushed, not pulled, and not present in the shipped image —
they exist only transiently in the local build cache (and can be pruned
with `docker builder prune`).

## Exercise

Convert a single-stage Python Dockerfile that does `pip install` from a
`requirements.txt` containing a compiled dependency (e.g. `psycopg2`,
which needs build headers) into two stages: a `builder` stage based on
`python:3.12` that installs build tools and produces wheels with
`pip wheel`, and a final stage based on `python:3.12-slim` that installs
those pre-built wheels with `pip install --no-index --find-links=/wheels`.
Compare `docker images` sizes before and after.
