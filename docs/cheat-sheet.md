---
title: "Docker Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Docker cheat sheet: 20 quick-reference tables from every level of the Docker Mastery Path, each linked to its full lesson."
---

# Docker Cheat Sheet

Quick-reference tables from **19 lessons** of the free Docker Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [01 · What Is Docker &amp; Why Containers?](level-1/01-what-is-docker.md)

*Containers vs virtual machines*

| | Virtual Machine | Container |
|---|---|---|
| Isolation unit | Full guest OS + kernel | Process, isolated via kernel features |
| Boot time | Seconds to minutes | Milliseconds to ~1 second |
| Image size | Gigabytes (whole OS) | Megabytes to low hundreds of MB |
| Overhead | A hypervisor + full OS per VM | Shares the host kernel |
| Density | Tens of VMs per host, typically | Hundreds of containers per host |
| Isolation strength | Very strong (separate kernel) | Weaker than a VM (shared kernel) |

### [02 · Installing Docker](level-1/02-installing-docker.md)

*Choosing what to install*

| Platform | Recommended install | Notes |
|---|---|---|
| macOS | **Docker Desktop** | Runs a lightweight Linux VM under the hood; provides the daemon, CLI, and a GUI |
| Windows | **Docker Desktop** (with WSL2 backend) | Needs WSL2 enabled; also runs Linux containers inside a Linux environment |
| Linux | **Docker Engine** (native packages) | Runs natively — no VM needed, since the host kernel *is* Linux |

### [03 · Images vs Containers](level-1/03-images-vs-containers.md)

*Cheat sheet*

| Command | Operates on |
|---|---|
| `docker images` / `docker image ls` | Images |
| `docker rmi <image>` | Images |
| `docker pull` / `docker push` | Images |
| `docker ps -a` / `docker container ls -a` | Containers |
| `docker run`, `docker start`, `docker stop` | Containers |
| `docker rm <container>` | Containers |

### [04 · Running Your First Container](level-1/04-first-container.md)

*Cheat sheet: the flags you'll use constantly*

| Flag | Meaning |
|---|---|
| `-d` | Run detached (in the background) |
| `-it` | Interactive + TTY (for a shell) |
| `--name <name>` | Give the container a fixed name |
| `-p host:container` | Publish a port to the host |
| `-e KEY=VALUE` | Set an environment variable |
| `--rm` | Auto-remove the container when it stops |
| `-v host:container` | Mount a volume or bind mount (module 07) |

### [05 · The Dockerfile &amp; Building Images](level-1/05-dockerfile-building-images.md)

*Part 1 · The Dockerfile: Other instructions worth knowing now*

| Instruction | Purpose |
|---|---|
| `ARG` | A build-time-only variable, available during `docker build` (via `--build-arg`), not present in the final running container unless also assigned to an `ENV` |
| `USER` | Switch to a non-root user for subsequent instructions and at runtime (covered in depth in Level 3's security module) |
| `VOLUME` | Declare a mount point intended for persistent/external data (module 07) |
| `LABEL` | Attach arbitrary key-value metadata to the image |

*Part 2 · Building Images: Common build errors and what they mean*

| Error | Cause |
|---|---|
| `COPY failed: file not found` | The file isn't inside the build context, or the path is wrong relative to it |
| `failed to solve: process ... did not complete successfully` | A `RUN` command exited non-zero (check the printed command output above the error) |
| `unable to prepare context: path ... does not exist` | You ran `docker build` from the wrong directory, or gave the wrong context path |
| Build is unexpectedly huge/slow | Missing a `.dockerignore`, so large directories like `.git` or `node_modules` are being sent and possibly copied |

### [06 · Container Lifecycle](level-1/06-container-lifecycle.md)

*The states*

| State | Meaning |
|---|---|
| **Created** | Filesystem and config exist; the main process has not started |
| **Running** | The main process is executing |
| **Paused** | Process execution is frozen (via cgroup freezer) but memory/state is retained |
| **Stopped / Exited** | The main process has exited (normally or via signal); filesystem still exists |
| **Removed** | Container and its writable layer are gone entirely |

### [07 · Volumes &amp; Bind Mounts](level-1/07-volumes-bind-mounts.md)

*Volumes vs bind mounts*

| | Volume | Bind mount |
|---|---|---|
| Managed by | Docker | You (arbitrary host path) |
| Location | Docker-controlled directory | Any host path you specify |
| Portability | Portable across hosts running Docker (path is abstracted) | Tied to that host's specific filesystem layout |
| Typical use | Databases, persistent application state | Local development (live-editing source code), injecting host config files |
| Referenced as | `-v <volume-name>:<container-path>` | `-v <host-absolute-path>:<container-path>` |

### [08 · Networking Basics](level-1/08-networking-basics.md)

*Other network drivers, briefly*

| Driver | What it does |
|---|---|
| `bridge` | The default; an isolated private network on a single host (covered above) |
| `host` | The container shares the host's network namespace directly — no isolation, no port mapping needed, container ports *are* host ports |
| `none` | The container gets no network interface at all beyond loopback |
| `overlay` | Spans multiple Docker hosts (used in Swarm/multi-host setups) — covered in Level 3 |

### [09 · Docker Hub &amp; Registries](level-1/09-docker-hub-registries.md)

*Official images vs user/organization images*

| Reference | Meaning |
|---|---|
| `nginx` | Shorthand for `docker.io/library/nginx` — an **official image**, maintained/reviewed by Docker Hub |
| `bitnami/nginx` | An image published under the `bitnami` user/org namespace |
| `myusername/myapp` | An image you publish under your own Docker Hub username |
| `ghcr.io/someorg/someimage` | An image hosted on GitHub Container Registry instead of Docker Hub |

### [10 · Project — Containerize a Small App](level-1/10-project-containerize-app.md)

*What this project proved*

| Requirement | How it was satisfied |
|---|---|
| A Dockerfile | `FROM`/`WORKDIR`/`COPY`/`RUN`/`EXPOSE`/`CMD`, ordered for caching |
| Build an image | `docker build -t visit-counter:1.0 .` |
| Run with a mounted volume | `-v visit-counter-data:/data`, verified to survive container removal |
| Run with a mapped port | `-p 8080:5000`, verified via `curl` |
| (Bonus) Registry round-trip | Tag, login, push |

## Level 2 · Intermediate

### [02 · Docker Compose Basics](level-2/02-docker-compose-basics.md)

*What each top-level key does*

| Key | Purpose |
|---|---|
| `services` | One entry per container Compose manages; the key (`web`, `db`) becomes both the container's name prefix and its hostname on the Compose network |
| `build` | Build an image from a local Dockerfile instead of pulling one |
| `image` | Pull (or reference) an existing image by name/tag |
| `ports` | `HOST:CONTAINER` port publishing, same semantics as `docker run -p` |
| `environment` | Environment variables injected into the container |
| `volumes` (service-level) | Bind mounts or named-volume mounts for that service |
| `volumes` (top-level) | Declares named volumes so Compose knows to create/manage them |
| `depends_on` | Controls **start order** only — it does not wait for the dependency to be *ready* (see module 06 for that) |
| `networks` | Custom networks beyond the default one Compose creates automatically |

### [03 · Environment Variables &amp; .env Files](level-2/03-environment-variables-env-files.md)

*Distinguishing three separate mechanisms*

| Mechanism | Where it applies | Purpose |
|---|---|---|
| `ENV` in Dockerfile | Baked into the image | Defaults that ship with the image itself |
| `environment:` / `-e` | Container runtime | Per-run/per-environment overrides |
| `.env` (Compose only) | Compose YAML parsing | Substituting values *into* the compose file text, before it's even sent to the Engine |

### [06 · Health Checks](level-2/06-health-checks.md)

*`HEALTHCHECK` in a Dockerfile*

| Option | Meaning |
|---|---|
| `--interval` | Time between checks (default 30s) |
| `--timeout` | How long a single check may run before it's considered failed |
| `--start-period` | Grace period after container start during which failures don't count toward `--retries` (for slow-starting apps) |
| `--retries` | Consecutive failures required before marking `unhealthy` |

### [08 · Managing Images &amp; Tags](level-2/08-managing-images-tags.md)

*Tagging strategies*

| Strategy | Example | Trade-off |
|---|---|---|
| Semantic version | `myapp:1.4.2` | Precise, reproducible, but you must remember to bump it |
| Git SHA | `myapp:a1b2c3d` | Always unique, traceable to exact source, unreadable to humans |
| `latest` | `myapp:latest` | Convenient default, but **mutable** — it silently points at whatever was last pushed, which is exactly why it should never be the tag a production deployment pins to |
| Environment | `myapp:staging`, `myapp:prod` | Readable, but also mutable — same caveat as `latest` |

## Level 3 · Advanced

### [01 · Docker Networking Deep Dive](level-3/01-networking-deep-dive.md)

*The drivers*

| Driver | Scope | Typical use |
|---|---|---|
| `bridge` (default, or user-defined) | Single host | Most containers on one machine; the default for `docker run` and Compose |
| `host` | Single host, no isolation | Maximum network performance, at the cost of no port-mapping isolation |
| `overlay` | Multiple hosts | Swarm services that need containers on different machines to talk to each other as if on one network |
| `none` | No networking at all | Fully isolated batch jobs that shouldn't touch the network |
| `macvlan` | Single host, direct L2 | Giving a container its own MAC/IP on the physical LAN, as if it were a separate physical machine |

### [02 · Security Basics](level-3/02-security-basics.md)

*Choosing a minimal base image*

| Base | Approx. size | Trade-off |
|---|---|---|
| `ubuntu:22.04` | ~78MB | Familiar tooling, but a larger attack surface (shell, package manager, many libraries) |
| `debian:12-slim` | ~30MB | Good middle ground |
| `alpine:3.19` | ~7MB | Very small, but uses `musl` libc instead of `glibc` — can surface subtle compatibility issues with prebuilt binaries |
| `gcr.io/distroless/*` | a few MB | No shell, no package manager at all — smallest attack surface, but harder to debug interactively |

### [03 · Resource Limits (CPU/Memory)](level-3/03-resource-limits.md)

*Memory limits*

| Flag | Meaning |
|---|---|
| `--memory` | Hard cap on RAM the container may use |
| `--memory-swap` | Cap on RAM **plus** swap combined; setting it equal to `--memory` disables swap for this container entirely |
| `--memory-reservation` | A soft limit — only enforced under host memory pressure, not a hard ceiling |

### [07 · Logging Drivers &amp; Centralized Logging](level-3/07-logging-drivers.md)

*Common drivers*

| Driver | Destination |
|---|---|
| `json-file` (default) | Local JSON-lines file, readable via `docker logs` |
| `local` | A more space-efficient local binary format, still readable via `docker logs` |
| `syslog` | A syslog daemon (local or remote) |
| `journald` | The systemd journal |
| `fluentd` | A Fluentd collector, typically forwarding onward to Elasticsearch/Loki/etc. |
| `awslogs` | Amazon CloudWatch Logs directly |
| `gelf` | Graylog/Logstash via the GELF protocol |
| `none` | Discard entirely — no log storage at all |

## Level 4 · Master

### [02 · Orchestration Handoff (Kubernetes/Swarm)](level-4/02-orchestration-handoff.md)

*Concept mapping: Swarm to Kubernetes*

| Swarm concept | Kubernetes equivalent | Key difference |
|---|---|---|
| Service (`docker service create`) | Deployment + Service | Kubernetes splits "desired replica set" (Deployment) from "stable network identity" (Service) into two objects |
| Task | Pod | A Pod can hold multiple co-scheduled containers sharing a network namespace (module 04 of Level 3's sidecar pattern, natively) |
| Overlay network | CNI network plugin | Kubernetes delegates networking to a pluggable CNI implementation rather than a single built-in driver |
| `docker secret` | Kubernetes `Secret` | Conceptually identical (mounted as files/env, module 09 of Level 3); Kubernetes secrets are base64-encoded by default, not encrypted, unless encryption-at-rest is separately configured |
| `docker config` | Kubernetes `ConfigMap` | Same purpose, same distinction from Secret |
| `docker stack deploy -c stack.yml` | `kubectl apply -f manifests/` | Both apply a declarative desired state; Kubernetes's object model is far larger (dozens of resource kinds vs. Compose's handful) |

---

Go deeper with the full [Docker Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
