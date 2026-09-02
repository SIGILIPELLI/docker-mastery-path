# 04 · Running Your First Container

With the concepts from the last two modules in place, this module is
hands-on: running containers, understanding `docker run`'s most important
flags, and reading their output.

## `docker run` end to end

```bash
docker run hello-world
```

When you run this, Docker performs a sequence of steps worth internalizing
because it explains almost every "why is this slow" or "why did it pull
from the internet" question later:

1. **Resolve the image** — parse `hello-world` as `hello-world:latest`.
2. **Check local cache** — look for that image (by name:tag) among images
   already pulled.
3. **Pull if missing** — if not cached, contact the registry (Docker Hub
   by default) and download the image's layers.
4. **Create a container** — create a new container from the image: a new
   writable layer, new namespaces (PID, network, mount, etc.), new
   cgroups for resource limits.
5. **Start the process** — run the image's configured command (its `CMD`
   or `ENTRYPOINT`) as PID 1 inside the container's PID namespace.
6. **Attach or detach** — by default `docker run` attaches your terminal
   to the container's stdout/stderr and streams its output until the
   process exits.

## Running interactively

```bash
docker run -it ubuntu bash
```

- `-i` (interactive) keeps STDIN open so you can type into the container.
- `-t` allocates a pseudo-TTY, so it behaves like a normal terminal
  (prompts, line editing, colors).
- Together, `-it` is what you want for "drop me into a shell inside a
  container."

Inside, you're now in an isolated filesystem based on the `ubuntu` image;
`exit` (or Ctrl-D) stops the container's main process, which stops the
container itself since a container's lifetime is tied to its PID 1
process.

## Running in the background

```bash
docker run -d --name my-nginx nginx
```

- `-d` (detached) starts the container and immediately returns your
  terminal, instead of streaming logs.
- `--name my-nginx` gives it a memorable name instead of a random one
  like `nginx_agitated_turing`, so you can reference it in later commands.

Check on it:

```bash
docker ps
# CONTAINER ID   IMAGE   COMMAND                  STATUS         NAMES
# 8f3a...        nginx   "nginx -g 'daemon of…"   Up 5 seconds   my-nginx

docker logs my-nginx
# streams nginx's stdout/stderr log lines
```

## Publishing a port

A container's network is isolated by default — nothing on the host can
reach a port inside the container unless you explicitly publish it:

```bash
docker run -d --name my-nginx -p 8080:80 nginx
```

`-p 8080:80` maps **host port 8080** to **container port 80** (the syntax
is always `-p <host_port>:<container_port>`). After this, visiting
`http://localhost:8080` from the host reaches nginx's port 80 inside the
container.

```bash
curl http://localhost:8080
# <html>...Welcome to nginx!...</html>
```

Without `-p`, nginx is running and listening on port 80 *inside its own
network namespace*, but that namespace isn't reachable from the host at
all.

## Passing environment variables

```bash
docker run -d --name my-db -e POSTGRES_PASSWORD=secret postgres:16
```

`-e KEY=VALUE` sets an environment variable inside the container — many
official images (like `postgres`) use environment variables as their
primary configuration mechanism. `-e` can be repeated for multiple
variables.

## Cleaning up automatically

```bash
docker run --rm alpine echo "hello"
# hello
```

`--rm` tells Docker to automatically delete the container (its writable
layer and metadata) as soon as it exits — useful for one-off commands
where you don't need to inspect the container afterward. Without `--rm`,
stopped containers stick around (visible via `docker ps -a`) until you
explicitly `docker rm` them.

## Cheat sheet: the flags you'll use constantly

| Flag | Meaning |
|---|---|
| `-d` | Run detached (in the background) |
| `-it` | Interactive + TTY (for a shell) |
| `--name <name>` | Give the container a fixed name |
| `-p host:container` | Publish a port to the host |
| `-e KEY=VALUE` | Set an environment variable |
| `--rm` | Auto-remove the container when it stops |
| `-v host:container` | Mount a volume or bind mount (module 08) |

## Worked example: a tiny web server, start to finish

```bash
docker run -d --name hello-web -p 8000:80 nginx
curl -s http://localhost:8000 | head -n 5
# <!DOCTYPE html>
# <html>
# <head>
# <title>Welcome to nginx!</title>
# </head>
docker logs hello-web
# 172.17.0.1 - - [.../GET / HTTP/1.1" 200 615 ...
docker stop hello-web
docker rm hello-web
```

This sequence — run detached with a published port, verify with `curl`,
inspect logs, then stop and remove — is the pattern you'll repeat
constantly while developing against containerized services.

## Exercise

Run an `nginx` container named `practice-web`, detached, publishing
container port 80 to host port 9090. Use `curl` (or your browser) to
confirm it serves the default nginx page. Then view its logs with
`docker logs practice-web`, stop it, and remove it. Finally, run
`docker run --rm alpine echo "cleanup works"` and confirm with
`docker ps -a` that no leftover container from that last command remains.
