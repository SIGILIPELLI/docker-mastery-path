---
description: "Storage Drivers & Volume Plugins — Every image layer and every container's writable layer has to be represented on disk somehow. The storage driver…"
---

# 06 · Storage Drivers & Volume Plugins

Every image layer and every container's writable layer has to be
represented on disk somehow. The **storage driver** decides how; a
**volume plugin** extends where volume data can physically live, beyond
the local host's filesystem.

## Checking your storage driver

```bash
docker info | grep "Storage Driver"
# Storage Driver: overlay2
```

`overlay2` is the modern default on Linux and the one covered throughout
this course's "How It Actually Works" sections; older/alternative
drivers (`aufs`, `devicemapper`, `btrfs`, `zfs`) exist for specific
kernel/filesystem combinations but overlay2 is the right default choice
for nearly all current Linux hosts.

## How overlay2 layers a filesystem, at the directory level

```bash
docker inspect -f '{{.GraphDriver.Data.MergedDir}}' mycontainer
docker inspect -f '{{.GraphDriver.Data.UpperDir}}' mycontainer
docker inspect -f '{{.GraphDriver.Data.LowerDir}}' mycontainer
```

- `LowerDir`: one or more read-only directories, one per image layer,
  stacked in order.
- `UpperDir`: the single writable directory unique to this container —
  everything the container writes lands here.
- `MergedDir`: the unified view the container's process actually sees as
  its root filesystem — overlayfs presents `LowerDir` + `UpperDir`
  merged into one coherent tree.

## Volume plugins for non-local storage

The built-in `local` volume driver stores data on the host's own disk.
A **volume plugin** implements Docker's volume driver API to back
volumes with something else entirely — network storage, cloud block
storage, distributed filesystems:

```bash
docker plugin install rexray/ebs
docker volume create --driver rexray/ebs --name my-ebs-volume
docker run -d --name app -v my-ebs-volume:/data myapp
```

From inside the container, `/data` looks like an ordinary local
directory — the plugin handles attaching/mounting the underlying cloud
volume transparently. This matters for scenarios plain host storage
can't provide: a volume that must survive the *host* being terminated
(common in autoscaled cloud fleets), or one shared concurrently across
multiple hosts.

## NFS as a built-in volume option (no plugin needed)

```bash
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/exported/path \
  nfs-volume

docker run -d -v nfs-volume:/data myapp
```

The `local` driver itself supports mounting NFS shares as "volumes"
without a third-party plugin, by shelling out to the same `mount`
options you'd otherwise use directly — a lighter-weight option than a
full plugin when NFS is the only extra capability you need.

## Worked example: inspecting layer sharing directly

```bash
docker run -d --name c1 python:3.12-slim sleep 3600
docker run -d --name c2 python:3.12-slim sleep 3600

docker inspect -f '{{.GraphDriver.Data.LowerDir}}' c1
docker inspect -f '{{.GraphDriver.Data.LowerDir}}' c2
# identical LowerDir chains — both containers share the exact same
# read-only image layers on disk; only their (separate) UpperDir differs
```

## How It Actually Works

**What overlayfs actually does at the kernel level.** `overlay2` is a
real Linux kernel filesystem (`mount -t overlay`), not a Docker
invention — Docker calls `mount` with `lowerdir=`, `upperdir=`, and
`workdir=` options for each container. The kernel then presents a
merged view where a file lookup checks `upperdir` first and falls back
through the `lowerdir` stack; a write to a file that only exists in a
lower (read-only) layer triggers **copy-up**: the kernel transparently
copies that file into `upperdir` first, then applies the write there —
the original lower-layer copy is untouched. This copy-up mechanism is
the literal, kernel-level explanation for why editing a large file that
originated in a base image (say, replacing one line in a multi-gigabyte
file shipped in `python:3.12`) is unexpectedly slow and grows the
container's writable layer by the *entire file's* size, not just the
size of your edit — the kernel had to copy the whole file up before it
could apply any part of the write.

**Why a whiteout file, not real deletion, represents "delete" across
layers.** Because lower layers are strictly read-only, `overlay2` can't
actually erase a file that lives in one. Deleting such a file instead
creates a special zero-size character device (major/minor number 0/0)
in `upperdir` with the same name — a "whiteout" — which the kernel's
overlay lookup logic recognizes and uses to hide the lower layer's
version from the merged view, without touching the lower layer's actual
bytes at all. This is the same mechanism referenced back in Level 1's
Dockerfile lesson (why `RUN rm` in a later layer doesn't shrink the
image) — the whiteout is created in that later layer, while the deleted
file's bytes remain physically present in whichever earlier layer
originally introduced them, still counted in the image's total size on
disk and during registry push/pull.

## Exercise

Start a container from any base image, `docker exec` into it and modify
a file that came from the base image (not one you `COPY`'d in), then
inspect `docker diff <container>` to see it reported as changed (`C`).
Cross-reference `.GraphDriver.Data.UpperDir` on the host (as root, if
needed) and confirm the modified file's *full* contents now exist there
via copy-up, even though you changed only a few bytes.
