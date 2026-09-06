# 11 · Project — Containerize a Small App

This capstone combines every Level 1 module into one project: writing a
Dockerfile, building an image, running it with a mounted volume *and* a
published port, and confirming everything with the commands you now know.

## The app

A tiny Python Flask app that counts how many times it's been visited,
persisting the count to a file so the count survives container restarts.

**`app.py`**

```python
import os
from flask import Flask

app = Flask(__name__)
COUNT_FILE = "/data/count.txt"

def read_count():
    if os.path.exists(COUNT_FILE):
        with open(COUNT_FILE) as f:
            return int(f.read().strip() or 0)
    return 0

def write_count(n):
    os.makedirs(os.path.dirname(COUNT_FILE), exist_ok=True)
    with open(COUNT_FILE, "w") as f:
        f.write(str(n))

@app.route("/")
def index():
    n = read_count() + 1
    write_count(n)
    return f"This page has been visited {n} time(s).\n"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**`requirements.txt`**

```
flask==3.0.3
```

## The Dockerfile

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

Notice the ordering from module 05/06: `requirements.txt` is copied and
installed *before* `app.py`, so editing the app later won't invalidate the
(slower) dependency-install layer.

## `.dockerignore`

```
__pycache__
*.pyc
.git
```

## Build the image

```bash
docker build -t visit-counter:1.0 .
docker images visit-counter
```

## Create a volume for persistent data

```bash
docker volume create visit-counter-data
```

This volume will hold `count.txt` at `/data` inside the container,
independent of the container's own lifecycle (module 08).

## Run it: mounted volume + published port

```bash
docker run -d \
  --name visit-counter \
  -p 8080:5000 \
  -v visit-counter-data:/data \
  --restart unless-stopped \
  visit-counter:1.0
```

This single command exercises the three requirements together:

- `-p 8080:5000` — publishes the container's Flask port (5000) to host
  port 8080 (module 04/09).
- `-v visit-counter-data:/data` — mounts the named volume so `count.txt`
  outlives the container (module 08).
- `--restart unless-stopped` — a lifecycle policy so it comes back up
  automatically after a host reboot or crash (module 07).

## Verify it end to end

```bash
curl http://localhost:8080
# This page has been visited 1 time(s).
curl http://localhost:8080
# This page has been visited 2 time(s).
curl http://localhost:8080
# This page has been visited 3 time(s).

docker logs visit-counter
```

Now prove the data survives the container being destroyed and recreated —
the core point of using a volume:

```bash
docker rm -f visit-counter

docker run -d \
  --name visit-counter \
  -p 8080:5000 \
  -v visit-counter-data:/data \
  visit-counter:1.0

curl http://localhost:8080
# This page has been visited 4 time(s).
```

The count continued from `4`, not reset to `1`, because the *volume* —
not the container — held the state, and the volume was reattached to the
brand-new container.

## Push it to a registry (optional, ties in module 10)

```bash
docker tag visit-counter:1.0 yourusername/visit-counter:1.0
docker login
docker push yourusername/visit-counter:1.0
```

## Clean up

```bash
docker rm -f visit-counter
docker volume rm visit-counter-data
docker rmi visit-counter:1.0
```

## What this project proved

| Requirement | How it was satisfied |
|---|---|
| A Dockerfile | `FROM`/`WORKDIR`/`COPY`/`RUN`/`EXPOSE`/`CMD`, ordered for caching |
| Build an image | `docker build -t visit-counter:1.0 .` |
| Run with a mounted volume | `-v visit-counter-data:/data`, verified to survive container removal |
| Run with a mapped port | `-p 8080:5000`, verified via `curl` |
| (Bonus) Registry round-trip | Tag, login, push |

## How It Actually Works

This project quietly exercises four separate kernel/filesystem mechanisms
working together — worth naming explicitly since they're each doing real
work behind a one-line command.

- **Layer caching during the build.** Docker builds the image one
  instruction at a time, and for each instruction it checks whether it
  already has a cached layer produced by the *same instruction run
  against the same parent layer* (matched by hashing the instruction
  string plus, for `COPY`/`ADD`, the checksum of the copied files'
  content). Because `COPY requirements.txt .` + `RUN pip install` come
  *before* `COPY app.py .`, editing `app.py` only invalidates the cache
  from that `COPY` onward — the `pip install` layer's cache entry still
  matches, so Docker reuses the existing layer bit-for-bit instead of
  re-running `pip install` and re-downloading Flask.
- **Overlayfs at runtime.** When the container starts, Docker doesn't
  copy the image's files — it mounts an **overlayfs** union filesystem:
  every read-only image layer stacked as `lowerdir`s, plus one writable
  `upperdir` created fresh for this container instance. Reading a file
  transparently falls through to whichever layer actually has it (top
  layer wins); writing a file uses **copy-up** — the file is copied from
  its lower, read-only layer into the container's own `upperdir` before
  being modified, leaving the original image layer completely untouched.
  This is why deleting the container and starting a new one from the same
  image gives you a pristine filesystem again — the new container gets
  its own empty `upperdir`.
- **Why the volume survives that.** `count.txt` under `/data` would
  otherwise live inside that ephemeral `upperdir` and vanish with
  `docker rm`. A named volume sidesteps overlayfs entirely: Docker uses a
  **bind mount** (via the mount namespace) to graft a directory that
  lives under Docker's own storage area (`/var/lib/docker/volumes/...` on
  the host) directly onto `/data` inside the container's mount namespace.
  That directory's lifecycle is managed independently of any container,
  which is the literal reason `docker rm -f visit-counter` followed by a
  fresh `docker run -v visit-counter-data:/data ...` reattaches the exact
  same inode-backed files and the counter resumes at 4 instead of resetting.
- **Port publishing.** As covered in module 09, `-p 8080:5000` installs an
  iptables DNAT rule mapping the host's port 8080 into this specific
  container's network namespace at port 5000 — `curl http://localhost:8080`
  never talks to Flask directly, it talks to the kernel's netfilter layer,
  which rewrites the packet's destination before delivery.
- **`--restart unless-stopped`.** This isn't monitored by the container's
  own process — the Docker daemon persists this policy in the container's
  metadata (`/var/lib/docker/containers/<id>/config.v2.json`) and, on
  daemon startup or whenever the container's process exits non-manually,
  consults that policy to decide whether to `start` it again — which is
  why it survives host reboots but not an explicit `docker stop`.

## Exercise (extend the project)

Add a second route, `/reset`, that deletes `count.txt` (resetting the
counter to zero on the next visit), rebuild the image with a new tag
(`visit-counter:1.1`), and run a fresh container from it using the *same*
`visit-counter-data` volume as before. Confirm that visiting `/` after
`/reset` starts counting from `1` again — this checks that you understand
both how to iterate on an image (module 06) and that the volume's content,
not the image version, determines the running state.
