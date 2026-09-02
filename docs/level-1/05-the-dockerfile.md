# 05 · The Dockerfile

A **Dockerfile** is a plain-text recipe: a sequence of instructions that
Docker executes, in order, to produce an image. Each instruction typically
adds one layer to the image.

## The core instructions

### `FROM` — pick a base image

```dockerfile
FROM python:3.12-slim
```

Every Dockerfile (except one built entirely `FROM scratch`) starts by
choosing a base image to build on top of. `FROM scratch` starts from a
completely empty filesystem — used mainly for statically compiled
binaries (e.g. a Go binary) that need nothing else.

### `WORKDIR` — set the working directory

```dockerfile
WORKDIR /app
```

Sets the directory that subsequent `RUN`, `CMD`, `COPY`, and `ADD`
instructions operate relative to, and creates it if it doesn't already
exist. Prefer this over `RUN cd /app && ...`, since `cd` in a `RUN` only
affects that one instruction's shell, not later instructions.

### `COPY` — bring files into the image

```dockerfile
COPY requirements.txt .
COPY . .
```

Copies files/directories from the **build context** (the directory you
pass to `docker build`, typically `.`) into the image's filesystem.
`COPY` is preferred over `ADD` for plain file copying — `ADD` has extra
behavior (auto-extracting tarballs, fetching URLs) that's rarely what you
want and can be surprising.

### `RUN` — execute a command while building

```dockerfile
RUN pip install --no-cache-dir -r requirements.txt
```

Executes a command inside a temporary container based on the current
layers, and commits the filesystem changes as a new layer. This is how
you install packages, compile code, or otherwise mutate the filesystem at
build time (as opposed to `CMD`, which only runs when a container starts).

### `ENV` — set environment variables

```dockerfile
ENV PYTHONUNBUFFERED=1
```

Sets an environment variable that persists into the running container
(and is visible to later `RUN` instructions too), unlike a shell
`export`, which wouldn't survive across separate `RUN` layers.

### `EXPOSE` — document a listening port

```dockerfile
EXPOSE 8000
```

`EXPOSE` is documentation/metadata — it does **not** publish the port to
the host by itself (that's what `-p` on `docker run` does). It signals to
humans and to tools like `docker run -P` (capital P, publish *all*
exposed ports to random host ports) which ports the containerized app
listens on.

### `CMD` — the default command

```dockerfile
CMD ["python", "app.py"]
```

Specifies the default command a container runs when started, unless
overridden by an argument to `docker run`. Use the **exec form**
(JSON array, as above) rather than the shell form (`CMD python app.py`)
so the process becomes PID 1 directly, without an extra shell process
wrapping it — this matters for correctly receiving signals like `SIGTERM`
on `docker stop`.

### `ENTRYPOINT` — a fixed command, with `CMD` as default arguments

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

## Other instructions worth knowing now

| Instruction | Purpose |
|---|---|
| `ARG` | A build-time-only variable, available during `docker build` (via `--build-arg`), not present in the final running container unless also assigned to an `ENV` |
| `USER` | Switch to a non-root user for subsequent instructions and at runtime (covered in depth in Level 3's security module) |
| `VOLUME` | Declare a mount point intended for persistent/external data (module 08) |
| `LABEL` | Attach arbitrary key-value metadata to the image |

## A complete, annotated Dockerfile

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

## Instruction execution order matters

Docker processes a Dockerfile top to bottom, and each instruction builds
on the filesystem state left by the previous one. `COPY . .` placed
*before* `RUN pip install ...` would work, but would also mean any change
to application code invalidates the cached dependency-install layer,
forcing a slow reinstall on every code change — see Level 2's caching
module for the fix (copying the dependency manifest first, as shown
above).

## Exercise

Write a Dockerfile for a tiny Node.js script:

- Base image: `node:20-slim`
- Working directory: `/usr/src/app`
- Copy `package.json` first, run `npm install`, then copy the rest of the
  source
- `EXPOSE 3000`
- Default command: `["node", "server.js"]` (exec form)

You don't need a real Node app yet to write the Dockerfile correctly —
building and running it comes in the next two modules.
