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
