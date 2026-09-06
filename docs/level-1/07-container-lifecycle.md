# 07 · Container Lifecycle

A container moves through a small set of states, and Docker gives you a
command for each transition. Understanding this state machine explains
why `docker ps` sometimes shows nothing when you expect a container, and
what "restarting" actually costs versus "removing and re-running."

## The states

```
        docker create           docker start
             │                       │
             ▼                       ▼
   (image) ──────────► [Created] ──────────► [Running] ──┐
                              ▲                    │      │ docker stop / kill
                              │ docker start        │      ▼
                              │ (of a stopped one)  │  [Stopped/Exited]
                              └──────────────────────      │
                                                            │ docker rm
                                                            ▼
                                                        (gone)
```

| State | Meaning |
|---|---|
| **Created** | Filesystem and config exist; the main process has not started |
| **Running** | The main process is executing |
| **Paused** | Process execution is frozen (via cgroup freezer) but memory/state is retained |
| **Stopped / Exited** | The main process has exited (normally or via signal); filesystem still exists |
| **Removed** | Container and its writable layer are gone entirely |

## The commands, one per transition

```bash
# Image -> Created (does not start it)
docker create --name demo nginx

# Created/Stopped -> Running
docker start demo

# Running -> Stopped (graceful: sends SIGTERM, waits, then SIGKILL)
docker stop demo

# Running -> Stopped (immediate SIGKILL, no grace period)
docker kill demo

# Running -> Paused / Paused -> Running
docker pause demo
docker unpause demo

# Anything -> gone (must be stopped first, unless -f is used)
docker rm demo
docker rm -f demo   # force: stop (kill) then remove in one step

# Restart: stop then start again, same container/filesystem
docker restart demo
```

`docker run` is shorthand for `docker create` followed immediately by
`docker start` (with `-a` to attach, unless `-d` is given) — knowing this
explains why `docker run --name x ...` fails the second time you run it
with the same name: the container named `x` already exists (in Created or
Exited state) and must be removed or started, not re-created.

## `docker stop` vs `docker kill`

`docker stop` sends `SIGTERM` to the container's PID 1, then waits a grace
period (10 seconds by default, configurable with `-t`), and only sends
`SIGKILL` if the process hasn't exited by then. This gives well-behaved
applications a chance to close database connections, flush buffers, or
finish in-flight requests. `docker kill` sends `SIGKILL` (or another
signal you specify with `-s`) immediately, with no grace period — useful
when a container is hung and won't respond to `SIGTERM`.

```bash
docker stop -t 30 demo   # give it up to 30s to shut down gracefully
docker kill demo          # no grace period
```

!!! note "Why PID 1 and exec-form CMD matter here"
    If a container's main process is a shell (from shell-form `CMD` like
    `CMD python app.py`) rather than the app itself (exec-form
    `CMD ["python", "app.py"]`), the shell — not your app — receives the
    `SIGTERM`, and many shells don't forward it. This is one of the
    concrete, testable reasons module 05 recommended exec form.

## Restart policies

Rather than manually restarting a crashed container, you can tell Docker
to do it automatically:

```bash
docker run -d --name resilient --restart unless-stopped nginx
```

| Policy | Behavior |
|---|---|
| `no` (default) | Never restart automatically |
| `on-failure[:N]` | Restart only if it exits with a non-zero code, up to N times |
| `always` | Always restart, even after a manual `docker stop` followed by daemon restart |
| `unless-stopped` | Like `always`, but won't restart if it was explicitly stopped before the daemon restarted |

## Inspecting a container's state

```bash
docker inspect demo
```

Prints a large JSON document describing everything about the container:
its config, mounts, network settings, and — most relevant here — a
`State` object with fields like `Status`, `Running`, `ExitCode`, and
`StartedAt`. You can filter it directly with `--format`:

```bash
docker inspect -f '{{.State.Status}}' demo
# running

docker inspect -f '{{.State.ExitCode}}' demo
# 0
```

An `ExitCode` of `0` means the process exited normally; nonzero usually
signals an error, and by convention `137` means it was killed via
`SIGKILL` (128 + signal number 9) — often a sign of `docker stop` timing
out, or an out-of-memory kill.

## Removing things in bulk

```bash
# Remove all stopped containers
docker container prune

# Remove everything unused: stopped containers, dangling images,
# unused networks, and build cache
docker system prune
```

`docker system prune` is destructive but scoped to *unused* resources —
running containers and images referenced by a running container are
never touched by it.

## Worked example

```bash
docker run -d --name flaky --restart on-failure:3 alpine sh -c "sleep 2 && exit 1"
sleep 12
docker ps -a --filter name=flaky
# shows Exited, having been restarted up to 3 times, then given up
docker logs flaky
docker rm flaky
```

This demonstrates `on-failure:N`: Docker retries a crashing container a
bounded number of times rather than looping forever, then leaves it
stopped for you to inspect.

## How It Actually Works

**`docker pause` uses the cgroup freezer, not a signal.** Unlike stop/kill,
pausing a container doesn't send any signal at all. It writes to the
cgroup freezer controller (`cgroup.freeze` under cgroup v2, or the older
`freezer.state` under v1) for that container's cgroup. The kernel then
stops scheduling every process in that cgroup entirely — they sit frozen
mid-instruction, still holding their memory and open file descriptors,
consuming zero CPU, invisible to the scheduler until `docker unpause`
writes the cgroup back to "thawed." This is why pause/unpause is
essentially instantaneous and loses no state, unlike stop/start which
actually terminates and later re-execs the process from scratch.

**Why removing a container requires it to be stopped first.** A
container's cgroup and network namespace can only be torn down once
nothing is using them — specifically, once the PID namespace's init
process (PID 1) has exited, the kernel automatically SIGKILLs every
remaining process in that namespace and the namespace itself becomes
reclaimable. `docker rm` on a running container is refused (without `-f`)
because removing the overlayfs upper directory and network veth pair out
from under a live process would corrupt state the process is actively
using; `-f` is really "kill first, then do the normal teardown."

**Where exit code 137 comes from.** By Unix convention, when a process is
terminated by a signal rather than exiting via `exit()`, the shell/kernel
reports its status as `128 + signal number`. `SIGKILL` is signal 9, so
`128 + 9 = 137`. That's why a `docker stop` that times out and falls back
to `SIGKILL`, or an out-of-memory kill (the kernel's OOM killer sends
`SIGKILL` to the largest/most-recently-offending process when a cgroup
hits its memory limit), both show up as exit code 137 in `docker inspect`
— it's the same generic "died by signal 9" encoding the Linux kernel uses
for any process, container or not.

## Exercise

Start an `nginx` container named `lifecycle-demo` with
`--restart on-failure:5`. Verify it's `Up` via `docker ps`. Then
`docker kill` it and immediately check `docker ps -a` — does Docker
restart it, and if so, how many times, based on the policy you set?
(Since `nginx` doesn't crash on its own, killing it manually simulates a
failure for this exercise.) Finally, clean it up with `docker rm -f
lifecycle-demo`.
