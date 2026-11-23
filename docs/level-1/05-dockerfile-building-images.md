---
description: "The Dockerfile & Building Images — Part 1: The Dockerfile; Part 2: Building Images."
---

# 05 · The Dockerfile & Building Images

This module covers the full path from recipe to image: first how a Dockerfile describes an image instruction by instruction, then how docker build turns it into an image, with layer caching, tags and build context.

## Part 1 · The Dockerfile

A **Dockerfile** is a plain-text recipe: a sequence of instructions that
Docker executes, in order, to produce an image. Each instruction typically
adds one layer to the image.

### The core instructions

#### `FROM` — pick a base image

```dockerfile
FROM python:3.12-slim
```

Every Dockerfile (except one built entirely `FROM scratch`) starts by
choosing a base image to build on top of. `FROM scratch` starts from a
completely empty filesystem — used mainly for statically compiled
binaries (e.g. a Go binary) that need nothing else.

#### `WORKDIR` — set the working directory

```dockerfile
WORKDIR /app
```

Sets the directory that subsequent `RUN`, `CMD`, `COPY`, and `ADD`
instructions operate relative to, and creates it if it doesn't already
exist. Prefer this over `RUN cd /app && ...`, since `cd` in a `RUN` only
affects that one instruction's shell, not later instructions.

#### `COPY` — bring files into the image

```dockerfile
COPY requirements.txt .
COPY . .
```

Copies files/directories from the **build context** (the directory you
pass to `docker build`, typically `.`) into the image's filesystem.
`COPY` is preferred over `ADD` for plain file copying — `ADD` has extra
behavior (auto-extracting tarballs, fetching URLs) that's rarely what you
want and can be surprising.

#### `RUN` — execute a command while building

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

Executes a command inside a temporary container based on the current
layers, and commits the filesystem changes as a new layer. This is how
you install packages, compile code, or otherwise mutate the filesystem at
build time (as opposed to `CMD`, which only runs when a container starts).

#### `ENV` — set environment variables

```dockerfile
ENV PYTHONUNBUFFERED=1
```

Sets an environment variable that persists into the running container
(and is visible to later `RUN` instructions too), unlike a shell
`export`, which wouldn't survive across separate `RUN` layers.

#### `EXPOSE` — document a listening port

```dockerfile
EXPOSE 8000
```

`EXPOSE` is documentation/metadata — it does **not** publish the port to
the host by itself (that's what `-p` on `docker run` does). It signals to
humans and to tools like `docker run -P` (capital P, publish *all*
exposed ports to random host ports) which ports the containerized app
listens on.

#### `CMD` — the default command

```dockerfile
CMD ["python", "app.py"]
```

Specifies the default command a container runs when started, unless
overridden by an argument to `docker run`. Use the **exec form**
(JSON array, as above) rather than the shell form (`CMD python app.py`)
so the process becomes PID 1 directly, without an extra shell process
wrapping it — this matters for correctly receiving signals like `SIGTERM`
on `docker stop`.

#### `ENTRYPOINT` — a fixed command, with `CMD` as default arguments

```dockerfile
ENTRYPOINT ["python", "app.py"]
CMD ["--help"]
```

`ENTRYPOINT` sets the command that always runs; `CMD` (when used together
with `ENTRYPOINT`) supplies default *arguments* to it, which
`docker run <image> <other-args>` can override. If only `CMD` is set (no
`ENTRYPOINT`), `docker run <image> <other-command>` replaces the whole
command instead of just the arguments. Use `ENTRYPOINT` when the image
should always run one specific program (like a CLI tool), and plain `CMD`
when the whole command should be easy to override.

### Other instructions worth knowing now

| Instruction | Purpose |
|---|---|
| `ARG` | A build-time-only variable, available during `docker build` (via `--build-arg`), not present in the final running container unless also assigned to an `ENV` |
| `USER` | Switch to a non-root user for subsequent instructions and at runtime (covered in depth in Level 3's security module) |
| `VOLUME` | Declare a mount point intended for persistent/external data (module 07) |
| `LABEL` | Attach arbitrary key-value metadata to the image |

### A complete, annotated Dockerfile

```dockerfile
# 1. Start from an official, slim Python base image
FROM python:3.12-slim

# 2. Set the working directory for everything that follows
WORKDIR /app

# 3. Copy only the dependency manifest first (see module 04 in Level 2
#    for why this ordering matters for build caching)
COPY requirements.txt .

# 4. Install dependencies as a distinct, cacheable layer
RUN pip install --no-cache-dir -r requirements.txt

# 5. Now copy the rest of the application code
COPY . .

# 6. Document the port the app listens on
EXPOSE 8000

# 7. Set an environment variable used by the app
ENV PYTHONUNBUFFERED=1

# 8. Default command when the container starts
CMD ["python", "app.py"]
```

### Instruction execution order matters

Docker processes a Dockerfile top to bottom, and each instruction builds
on the filesystem state left by the previous one. `COPY . .` placed
*before* `RUN pip install ...` would work, but would also mean any change
to application code invalidates the cached dependency-install layer,
forcing a slow reinstall on every code change — see Level 2's caching
module for the fix (copying the dependency manifest first, as shown
above).

### How It Actually Works

Two Dockerfile behaviors that look like stylistic preferences are actually
consequences of how Linux processes and signals work.

**Why `RUN` produces a layer, mechanically.** Each `RUN` instruction
during `docker build` starts an ephemeral container from the current
image state (same `clone()`/namespace/overlayfs machinery as `docker
run`), executes the command inside it, then — instead of discarding the
container — the builder diffs its writable layer against what it started
with and commits *that diff* as a new, permanent, content-addressed image
layer (identified by hashing its contents). The ephemeral container is
then removed, but its filesystem delta lives on as a layer. This is why
`RUN rm -rf /some/big/file` in a *later* instruction doesn't shrink the
image: the file still physically exists in the earlier layer on disk —
deleting it later just adds a whiteout file in a new layer that hides it
from view, exactly like deleting a file inside a running container (see
module 03). The only way to actually not have those bytes in the image is
to not create them in that layer in the first place, or to use a
multi-stage build (Level 2) to copy only what you need into a fresh image.

**Exec form vs. shell form and PID 1 signals.** `CMD ["python", "app.py"]`
(exec form) causes the container runtime to `exec()` that binary directly
as PID 1. `CMD python app.py` (shell form) is silently rewritten to `CMD
["/bin/sh", "-c", "python app.py"]` — meaning `/bin/sh` becomes PID 1, and
your actual `python` process runs as its *child*, PID 2 or later. This
matters because `docker stop` sends `SIGTERM` to PID 1 specifically. A
shell interpreting `-c "python app.py"` does not automatically forward
signals to child processes it spawned — many shells simply ignore
`SIGTERM` while waiting on a child — so `python app.py` never receives it,
and Docker falls back to `SIGKILL` after the default 10-second grace
period, giving your process no chance to shut down cleanly (flush
buffers, close database connections, finish in-flight requests). The exec
form avoids the extra shell layer entirely, so your process *is* PID 1 and
receives `SIGTERM` directly.

### Exercise

Write a Dockerfile for a tiny Node.js script:

- Base image: `node:20-slim`
- Working directory: `/usr/src/app`
- Copy `package.json` first, run `npm install`, then copy the rest of the
  source
- `EXPOSE 3000`
- Default command: `["node", "server.js"]` (exec form)

You don't need a real Node app yet to write the Dockerfile correctly —
building and running it comes in the next two modules.

## Part 2 · Building Images

Once you have a Dockerfile, `docker build` turns it into an image. This
module covers the build command, tagging, and reading build output.

### The basic build command

```bash
docker build -t hello-app:1.0 .
```

- `-t hello-app:1.0` **tags** the resulting image with a repository name
  (`hello-app`) and a version tag (`1.0`). You can pass `-t` multiple
  times to apply several tags to the same build.
- `.` is the **build context**: the directory whose contents Docker sends
  to the daemon and which `COPY`/`ADD` instructions can reference. Docker
  cannot `COPY` a file that lives outside the build context.
- By default, Docker looks for a file literally named `Dockerfile` in the
  build context; use `-f <path>` to point at a differently named or
  located file.

### What actually happens during a build

```
docker build -t hello-app:1.0 .
```

1. The CLI packages up the build context (the directory tree at `.`,
   minus anything excluded by `.dockerignore`) and sends it to the Docker
   daemon.
2. The daemon parses the Dockerfile into a sequence of steps.
3. For each instruction, the daemon checks whether a cached layer already
   exists for that exact instruction plus the layer it builds on (see
   Level 2's caching module) — if so, it reuses the cache; if not, it
   executes the instruction in a temporary container and commits the
   result as a new layer.
4. Once every instruction is processed, the final set of layers plus
   metadata (default `CMD`, `EXPOSE`d ports, etc.) is assembled into an
   image and stored locally under the tag(s) you gave it.

Typical output looks like a numbered list of steps:

```
[+] Building 12.4s (10/10) FINISHED
 => [internal] load build definition from Dockerfile
 => [internal] load .dockerignore
 => [internal] load metadata for docker.io/library/python:3.12-slim
 => [1/5] FROM docker.io/library/python:3.12-slim
 => [internal] load build context
 => [2/5] WORKDIR /app
 => [3/5] COPY requirements.txt .
 => [4/5] RUN pip install --no-cache-dir -r requirements.txt
 => [5/5] COPY . .
 => exporting to image
 => => naming to docker.io/library/hello-app:1.0
```

(This output style reflects Docker's BuildKit builder, which has been the
default builder in current Docker versions; older Docker releases show a
simpler `Step 1/5 : FROM ...` style log instead — the instructions and
layers are the same either way.)

### The `.dockerignore` file

Just like `.gitignore`, a `.dockerignore` file in the build context
excludes files/directories from being sent to the daemon at all:

```
.git
node_modules
__pycache__
*.log
.env
```

This matters for two reasons: it keeps the build context small (faster
builds, especially over a remote daemon), and it prevents accidentally
`COPY .`-ing secrets, local virtual environments, or dependency
directories that should be reinstalled fresh inside the image rather than
copied from the host.

### Tagging images

```bash
docker build -t hello-app:1.0 -t hello-app:latest .
```

A single build can be tagged multiple times; both tags point at the same
image ID. Common conventions:

- A semantic version tag (`1.0`, `1.2.3`) for a specific, reproducible
  build.
- `latest` as a convenience alias for "the most recently built/pushed
  version" — but remember from module 03 that `latest` is just a tag
  like any other, not a guarantee of freshness or stability.
- A tag matching a git commit SHA (`hello-app:a1b2c3d`) for exact
  traceability from image back to source in CI pipelines.

### Verifying and inspecting the result

```bash
docker images hello-app
# REPOSITORY   TAG      IMAGE ID       CREATED          SIZE
# hello-app    1.0      7e2f...        10 seconds ago   135MB

docker run --rm hello-app:1.0
```

`docker history` shows the layer-by-layer size breakdown, which is useful
for spotting which instruction bloated the image:

```bash
docker history hello-app:1.0
# IMAGE          CREATED BY                                      SIZE
# 7e2f...        CMD ["python" "app.py"]                         0B
# <missing>      COPY . .                                        2.1kB
# <missing>      RUN pip install --no-cache-dir -r requiremen…   28.4MB
# <missing>      COPY requirements.txt .                         31B
# <missing>      WORKDIR /app                                    0B
# <missing>      FROM python:3.12-slim                           130MB
```

### Common build errors and what they mean

| Error | Cause |
|---|---|
| `COPY failed: file not found` | The file isn't inside the build context, or the path is wrong relative to it |
| `failed to solve: process ... did not complete successfully` | A `RUN` command exited non-zero (check the printed command output above the error) |
| `unable to prepare context: path ... does not exist` | You ran `docker build` from the wrong directory, or gave the wrong context path |
| Build is unexpectedly huge/slow | Missing a `.dockerignore`, so large directories like `.git` or `node_modules` are being sent and possibly copied |

### Worked example: building the Node Dockerfile from Part 1

Given the Dockerfile from the previous exercise, and a minimal
`package.json` + `server.js` in the same directory:

```bash
docker build -t node-hello:1.0 .
docker images node-hello
docker run --rm -p 3000:3000 node-hello:1.0
```

If the build succeeds and `docker run` starts without error, the image is
correctly structured; if `npm install` fails inside the build, the error
output will point at that `RUN` step specifically, since BuildKit reports
which step failed.

### How It Actually Works

**Build cache keys, precisely.** BuildKit doesn't decide "reuse this
layer" by comparing your Dockerfile text to a previous run in a fuzzy way
— it computes a **cache key per instruction** from two inputs: the parent
layer's content hash, and a normalized representation of the instruction
itself (the command string for `RUN`, or a hash of the *actual file
contents being copied* for `COPY`/`ADD`, not just the filename). If either
input changes, the cache key changes, and every subsequent instruction's
cache key changes too, because each one's key incorporates the previous
layer's hash — this is exactly why reordering `COPY requirements.txt .`
before `COPY . .` matters: `COPY .` changes its cache key on *any* source
file edit, so if it comes before `RUN pip install`, that install step's
cache key changes too and reruns, even though the actual dependency
manifest didn't change.

**What an image actually is on disk/registry.** An image is not one
opaque blob — it's a JSON **manifest** listing an ordered array of layer
digests (each a `sha256:...` of a compressed tarball) plus a reference to
a separate **image config** JSON object (environment variables, `CMD`,
`ENTRYPOINT`, exposed ports, architecture). `docker build` assembles this
manifest+config+layers set locally; `docker images`'s `IMAGE ID` is the
hash of the config JSON, while each layer keeps its own independent
content hash. `docker history` is reading that manifest's layer list back
out and matching sizes to instructions — `<missing>` appears for layers
that came from a pulled base image rather than being built locally in
this session, since Docker doesn't always retain per-instruction metadata
for layers it didn't just build.

### Exercise

Using the Dockerfile you wrote in Part 1 (or the Python one shown
above), run a build tagging the image with **two** tags at once — a
version number and `latest` — then use `docker images` to confirm both
tags point at the same `IMAGE ID`. Then add a `.dockerignore` excluding
at least `__pycache__` (or `node_modules`) and rebuild, confirming the
build still succeeds.
