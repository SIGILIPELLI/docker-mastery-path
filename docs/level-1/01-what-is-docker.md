# 01 · What Is Docker & Why Containers?

Docker is a platform for packaging an application together with everything
it needs to run — code, runtime, system libraries, configuration — into a
single, portable unit called a **container image**. Running that image
produces a **container**: an isolated process (or group of processes) on
the host machine's kernel.

The core promise is: *"it works on my machine"* becomes *"it works
everywhere the container runs,"* because the container carries its own
filesystem and dependencies instead of relying on whatever happens to be
installed on the host.

## Containers vs virtual machines

Both containers and virtual machines (VMs) solve the same underlying
problem — isolating and packaging workloads — but at different layers of
the stack.

| | Virtual Machine | Container |
|---|---|---|
| Isolation unit | Full guest OS + kernel | Process, isolated via kernel features |
| Boot time | Seconds to minutes | Milliseconds to ~1 second |
| Image size | Gigabytes (whole OS) | Megabytes to low hundreds of MB |
| Overhead | A hypervisor + full OS per VM | Shares the host kernel |
| Density | Tens of VMs per host, typically | Hundreds of containers per host |
| Isolation strength | Very strong (separate kernel) | Weaker than a VM (shared kernel) |

A VM runs a **hypervisor** (like VMware, Hyper-V, or KVM) that emulates
hardware, and each VM boots its own full operating system kernel on top of
that virtual hardware. A container, by contrast, is just a normal process
on the host, made to *believe* it has its own filesystem, process tree,
network stack, and hostname — using Linux kernel features:

- **Namespaces** — give a process its own isolated view of PIDs, network
  interfaces, mount points, hostname, and users, so it can't see (or
  interfere with) other processes' resources.
- **Control groups (cgroups)** — limit and account for how much CPU,
  memory, disk I/O, and network bandwidth a process (or group of
  processes) may consume.
- **A layered filesystem (union filesystem)** — lets a container's
  filesystem be built as a stack of read-only image layers plus one
  writable layer on top, so multiple containers can share the same
  underlying image layers on disk without duplicating them.

Because there's no second kernel to boot and no hardware to emulate, a
container starts about as fast as launching a regular process, and many
containers can run on a host that could support only a handful of VMs.

!!! note "On macOS and Windows"
    Linux containers need a Linux kernel. Docker Desktop on macOS and
    Windows runs a small Linux VM behind the scenes (via HyperKit/Virtualization.framework
    on macOS, WSL2 on Windows) and your containers actually run inside that
    VM — this is transparent to you, but it's why "Docker containers share
    the host kernel" is precisely true only on Linux hosts.

## Why this matters in practice

- **Consistency** — the same image runs identically on a developer's
  laptop, in CI, and in production, because the environment travels with
  the application.
- **Isolation without the VM tax** — each service can have its own
  dependency versions (a different Python or Node version, different
  library versions) without polluting the host or conflicting with other
  services.
- **Fast, cheap scaling** — spinning up another container to handle more
  load is fast and lightweight compared to booting another VM.
- **A standard packaging format** — a Docker image is a well-defined,
  portable artifact you can push to a registry, version with tags, and
  pull down anywhere Docker (or a compatible runtime) is installed.

## The pieces of Docker

| Component | What it does |
|---|---|
| **Docker Engine (`dockerd`)** | Background daemon that builds images and runs/manages containers on the host |
| **Docker CLI (`docker`)** | The command-line client you talk to; sends commands to the daemon over an API |
| **Image** | A read-only template (filesystem + metadata) used to create containers |
| **Container** | A running (or stopped) instance created from an image |
| **Dockerfile** | A text recipe describing how to build an image |
| **Registry** (e.g. Docker Hub) | A server that stores and distributes images by name and tag |

A typical workflow: you write a **Dockerfile**, `docker build` turns it
into an **image**, `docker run` starts a **container** from that image, and
`docker push`/`docker pull` move the image to and from a **registry** so
other machines can run it too.

## Worked example: same idea, two worlds

Imagine shipping a small Python script that needs Python 3.12 and one
library. Without containers, you'd write install instructions ("install
Python 3.12, then `pip install requests`, then run `python app.py`") and
hope every machine follows them correctly. With Docker, you instead ship
one artifact:

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app.py .
RUN pip install --no-cache-dir requests
CMD ["python", "app.py"]
```

Anyone with Docker installed runs the exact same environment with:

```bash
docker build -t hello-app .
docker run hello-app
```

No "which Python version do you have," no "did you forget to install
`requests`" — the image *is* the environment. We'll unpack `FROM`, `COPY`,
`RUN`, and `CMD` in detail in [module 05](05-the-dockerfile.md); for now,
notice that the whole environment is described in five lines and produces
one portable image.

## How It Actually Works

"A container is just a process with a different view of the system" is
true, but it's worth seeing exactly which kernel mechanisms create that
view — because none of it is magic and none of it is Docker-specific.

**Namespaces: what a process can see.** When the container runtime starts
a container, it doesn't launch a special "container process" — it calls
the same `clone()` system call any process uses to fork, but passes extra
flags telling the kernel to give the new process fresh, empty instances of
particular namespaces instead of inheriting the parent's:

| Namespace | Flag | Isolates |
|---|---|---|
| PID | `CLONE_NEWPID` | Process IDs — the container's first process becomes PID 1 inside its own PID tree, even though the host sees it as, say, PID 48213 |
| NET | `CLONE_NEWNET` | Network interfaces, routing tables, ports — the container gets its own loopback and virtual ethernet interface |
| MNT | `CLONE_NEWNS` | Mount points — the container sees its own root filesystem, not the host's |
| UTS | `CLONE_NEWUTS` | Hostname and domain name |
| IPC | `CLONE_NEWIPC` | System V IPC and POSIX message queues |
| USER | `CLONE_NEWUSER` | User/group ID mappings — root inside the container can map to an unprivileged UID outside it |

Each namespace is a separate, independently-toggleable kernel data
structure. `docker run` asks the kernel for all of them at once, which is
*why* a container process can't see host PIDs, can't bind to host network
interfaces directly, and sees `/` as its own filesystem root rather than
the host's `/`.

**cgroups: what a process can consume.** Namespaces control visibility,
not resource usage — a namespaced process could still consume 100% of the
host's RAM. Control groups (cgroups, implemented via the `cgroupfs`
pseudo-filesystem, typically under `/sys/fs/cgroup/`) are a separate
kernel mechanism that caps and accounts for CPU shares, memory, block I/O,
and PIDs for a group of processes. When you later pass flags like
`--memory` or `--cpus` to `docker run`, the daemon is writing numbers into
cgroup control files (e.g. `memory.max`) for that container's cgroup —
there's no separate "Docker resource manager," just files in a
special-purpose filesystem that the kernel enforces directly.

**Why the kernel is shared.** All of this — namespaces and cgroups alike —
are *views and limits applied to processes running on one kernel*. There's
only one kernel scheduler, one kernel memory manager, one set of loaded
kernel modules, shared by the host and every container on it. That's the
mechanical reason containers boot in milliseconds (no second kernel to
initialize) and why "container escape" vulnerabilities are a real category
of security bug: break out of your namespace/cgroup view and you're
talking to the same kernel everything else on the host uses.

## Exercise

Without running anything yet, write down (in a text file or scratch note)
answers to these three questions, in your own words:

1. If a container shares the host's kernel, what actually makes two
   containers on the same host unable to see each other's processes?
2. Why would a container typically start faster than a virtual machine?
3. Name one situation where you'd still reach for a full VM instead of a
   container (hint: think about isolation strength, or running a
   completely different kernel/OS).

You'll be able to check your reasoning against the concepts above — there's
no single "correct" wording, but a good answer references namespaces/cgroups
for (1) and (2), and kernel-level isolation for (3).
