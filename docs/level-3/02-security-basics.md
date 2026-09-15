---
description: "Security Basics — Three habits eliminate most of the container-security mistakes seen in practice: don't run as root inside the container, start from the…"
---

# 02 · Security Basics

Three habits eliminate most of the container-security mistakes seen in
practice: don't run as root inside the container, start from the
smallest base image that works, and scan images for known
vulnerabilities before shipping them.

## Running as a non-root user

By default, a container's main process runs as `root` *inside* the
container's user namespace — which, without additional isolation, maps
to the same UID as the host's root unless user-namespace remapping is
configured. A process that shouldn't need root privileges (most web
apps, workers, APIs) should not run as root even inside the container.

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY . .

# Create an unprivileged user and switch to it for the rest of the build
# and for the runtime process
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

CMD ["node", "src/server.js"]
```

Many official images already ship a non-root user for exactly this
purpose (e.g. `node` images include a `node` user):

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY --chown=node:node . .
USER node
CMD ["node", "src/server.js"]
```

`--chown` on `COPY` sets ownership at copy time, avoiding a separate
`RUN chown -R` layer that would otherwise duplicate the copied files'
size in the layer diff.

## Choosing a minimal base image

| Base | Approx. size | Trade-off |
|---|---|---|
| `ubuntu:22.04` | ~78MB | Familiar tooling, but a larger attack surface (shell, package manager, many libraries) |
| `debian:12-slim` | ~30MB | Good middle ground |
| `alpine:3.19` | ~7MB | Very small, but uses `musl` libc instead of `glibc` — can surface subtle compatibility issues with prebuilt binaries |
| `gcr.io/distroless/*` | a few MB | No shell, no package manager at all — smallest attack surface, but harder to debug interactively |

Smaller isn't automatically better for every situation — `alpine`'s musl
libc has tripped up compiled dependencies that assume glibc, and a fully
distroless image can't be `docker exec`'d into with a shell for
debugging. Choose based on your actual runtime needs, not size alone.

## Scanning images for known vulnerabilities

```bash
docker scout cveS myapp:latest
```

`docker scout` (bundled with recent Docker Desktop/CLI) checks an
image's installed packages against vulnerability databases and reports
known CVEs by severity, including which layer introduced them —
directing your fix to "upgrade this base image" or "update this
dependency" rather than a vague warning. Third-party equivalents (Trivy,
Grype) do the same job and are commonly wired into CI.

```bash
trivy image myapp:latest
```

## Dropping Linux capabilities

Even as non-root, a container process by default retains a set of Linux
**capabilities** (fine-grained slices of what used to be all-or-nothing
root privilege). Dropping ones the app doesn't need shrinks what an
exploited process could do:

```bash
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp
```

`--cap-drop=ALL` removes every capability, then `--cap-add=NET_BIND_SERVICE`
re-adds only the one needed to bind a port below 1024 — a common
combination for an app that must listen on, say, port 80 but has no
other reason to hold elevated Linux capabilities.

## Worked example: a hardened Dockerfile end to end

```dockerfile
FROM node:20-slim AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
USER nonroot
EXPOSE 8000
CMD ["src/server.js"]
```

```bash
docker run --read-only --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  -p 8000:8000 myapp:hardened
```

`--read-only` makes the container's root filesystem read-only at the
kernel level (module 03/Level 4 module 03 cover this and its
implications for apps that need a writable `/tmp`).

## How It Actually Works

**Why running as root inside a container is meaningfully risky even
though namespaces isolate it.** A container's PID/mount/network
namespaces isolate *what the process can see and name*, but by default
they do not remap UID 0 inside the container to a different UID outside
it — root inside is the same root the kernel has always known, UID 0,
just operating within a restricted view. If a container-escape
vulnerability (a kernel bug, a misconfigured mount, a Docker socket
exposed into the container) lets a process step outside its namespace
boundary, whether it lands with root or unprivileged permissions on the
host is decided by that UID, not by the namespace boundary that failed.
Running as a non-root UID inside the container means a successful escape
still hands the attacker only an unprivileged host UID, containing the
blast radius of that specific class of failure — user-namespace
remapping (mapping container UID 0 to an unprivileged host UID
project-wide) hardens this further but is a separate, host-level daemon
configuration.

**What a Linux capability actually is, mechanically.** Traditional Unix
had exactly two privilege levels for security checks: UID 0 (root, can
do anything) and everything else (subject to normal permission checks).
Capabilities split "anything" into about 40 independent bits stored in
each process's kernel-tracked credential structure — `CAP_NET_BIND_SERVICE`
(bind ports < 1024), `CAP_SYS_ADMIN` (a notoriously broad grab-bag),
`CAP_CHOWN`, and so on — and the kernel checks the specific bit relevant
to each privileged operation rather than a single root/non-root flag.
`--cap-drop=ALL --cap-add=X` sets a container's process capability set
to have every bit cleared except the one(s) explicitly re-added,
enforced by the kernel on every relevant syscall for that process's
lifetime — this is real kernel-level access control, not an
application-level convention Docker is layering on top.

## Exercise

Take any Dockerfile from an earlier module that doesn't set `USER`, add
a non-root user, run it with `--cap-drop=ALL`, and observe what (if
anything) breaks — a service binding a low port will fail with a
permission error unless you also `--cap-add=NET_BIND_SERVICE`, which is
the intended lesson: capabilities should be granted deliberately, one at
a time, based on what actually failed.
