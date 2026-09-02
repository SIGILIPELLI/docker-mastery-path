# 06 · Building Images

Once you have a Dockerfile, `docker build` turns it into an image. This
module covers the build command, tagging, and reading build output.

## The basic build command

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

## What actually happens during a build

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

## The `.dockerignore` file

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

## Tagging images

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

## Verifying and inspecting the result

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

## Common build errors and what they mean

| Error | Cause |
|---|---|
| `COPY failed: file not found` | The file isn't inside the build context, or the path is wrong relative to it |
| `failed to solve: process ... did not complete successfully` | A `RUN` command exited non-zero (check the printed command output above the error) |
| `unable to prepare context: path ... does not exist` | You ran `docker build` from the wrong directory, or gave the wrong context path |
| Build is unexpectedly huge/slow | Missing a `.dockerignore`, so large directories like `.git` or `node_modules` are being sent and possibly copied |

## Worked example: building the Node Dockerfile from module 05

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

## Exercise

Using the Dockerfile you wrote in module 05 (or the Python one shown
above), run a build tagging the image with **two** tags at once — a
version number and `latest` — then use `docker images` to confirm both
tags point at the same `IMAGE ID`. Then add a `.dockerignore` excluding
at least `__pycache__` (or `node_modules`) and rebuild, confirming the
build still succeeds.
