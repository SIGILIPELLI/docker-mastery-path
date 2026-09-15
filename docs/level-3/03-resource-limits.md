---
description: "Resource Limits (CPU/Memory) — Without limits, a single misbehaving container can consume all of a host's CPU or memory and starve every other container…"
---

# 03 · Resource Limits (CPU/Memory)

Without limits, a single misbehaving container can consume all of a
host's CPU or memory and starve every other container on it. Docker
exposes cgroup-backed flags to cap what a container may use.

## Memory limits

```bash
docker run -d --memory=512m --memory-swap=512m myapp
```

| Flag | Meaning |
|---|---|
| `--memory` | Hard cap on RAM the container may use |
| `--memory-swap` | Cap on RAM **plus** swap combined; setting it equal to `--memory` disables swap for this container entirely |
| `--memory-reservation` | A soft limit — only enforced under host memory pressure, not a hard ceiling |

Exceeding `--memory` doesn't throttle the process — the kernel's
out-of-memory killer terminates it, which is why a container that dies
with exit code `137` and no application-level error in its logs is a
strong signal to check whether it hit a memory limit
(`docker inspect -f '{{.State.OOMKilled}}' myapp`).

## CPU limits

```bash
docker run -d --cpus=1.5 myapp
```

`--cpus=1.5` caps the container to the equivalent of one and a half CPU
cores' worth of processing time, averaged over each scheduling period —
it can burst higher briefly and get throttled back down, rather than
being pinned to specific cores.

```bash
docker run -d --cpu-shares=512 myapp   # relative weight, not an absolute cap
```

`--cpu-shares` is fundamentally different from `--cpus`: it only matters
when the host is under CPU contention, at which point containers get CPU
time proportional to their share weight relative to others' — a
container with `--cpu-shares=512` gets half the CPU time of one with
`1024`, but *only* when both are competing for the same scarce CPU; if
the host is otherwise idle, a low-share container can still use 100% of
a core.

```bash
docker run -d --cpuset-cpus="0,1" myapp   # pin to specific physical cores
```

## Worked example: observing a memory limit being enforced

```bash
docker run -d --name mem-test --memory=100m alpine \
  sh -c "cat /dev/zero | head -c 200m | tail; sleep 30"

docker wait mem-test        # blocks until it exits, prints the exit code
docker inspect -f '{{.State.OOMKilled}}' mem-test   # true
docker logs mem-test         # likely empty or truncated — killed mid-write
```

## Setting limits in Compose

```yaml
services:
  worker:
    build: .
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 512M
        reservations:
          cpus: "0.25"
          memory: 128M
```

Under plain `docker compose up` (not Swarm mode), the `deploy.resources`
block is honored for `limits` since Compose v2 — no separate orchestrator
required for a single-host `docker compose` project to apply CPU/memory
caps.

## Monitoring against limits

```bash
docker stats --no-stream
```

`docker stats` prints live CPU%, memory usage, and — critically — the
memory **limit** it's calculated against, so you can see at a glance how
close a container is running to its own ceiling before it gets OOM
killed.

## How It Actually Works

**Where the numbers actually live: cgroups, not Docker itself.** Every
container gets its own **cgroup** (control group) — a kernel accounting
and limiting mechanism, unrelated to namespaces, that groups a set of
processes for the purpose of enforcing resource limits. `--memory=512m`
mechanically writes `512m` (converted to bytes) to that cgroup's
`memory.max` file (cgroup v2; `memory.limit_in_bytes` under v1) under
`/sys/fs/cgroup/`. The kernel's memory accounting subsystem then tracks
every page the container's processes allocate against that cgroup's
counter, and when an allocation would push the counter past `memory.max`,
the kernel's OOM killer is invoked *scoped to that cgroup* — it selects
and `SIGKILL`s the largest (or otherwise highest-scored) process within
that specific cgroup, leaving every process outside it, including other
containers and the host itself, completely unaffected. Docker's
`--memory` flag is therefore not an application-level convention at all
— it's a thin wrapper writing directly to kernel accounting files that
the kernel itself enforces during page allocation.

**Why `--cpus` produces throttling rather than a hard multi-core
pin.** `--cpus=1.5` is implemented via the CFS (Completely Fair
Scheduler) bandwidth controller: the cgroup gets a `cpu.max` setting of
"150000 100000" (cgroup v2 syntax) meaning "150,000 microseconds of CPU
time allowed per 100,000-microsecond period, across however many cores
the process actually runs on." The scheduler tracks cumulative runtime
for all threads in that cgroup within each 100ms period; once the
1.5-core-equivalent budget for that period is exhausted, every thread in
the cgroup is taken off the run queue (throttled) until the next period
begins, regardless of whether other cores sit completely idle in the
meantime. This period-based bursting explains a common confusion:
`docker stats` briefly showing CPU% well above the "expected" 150%
average for part of a period, followed by a stall, rather than a smooth
150% ceiling at all times.

## Exercise

Run a CPU-bound container (`docker run -d --cpus=0.5 alpine sh -c "yes >
/dev/null"`) and watch `docker stats` for it — confirm its CPU% settles
around 50%, not 100%, even though `yes` alone would happily consume an
entire core. Then remove the `--cpus` flag and rerun, comparing the CPU%
you observe.
