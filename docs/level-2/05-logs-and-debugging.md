---
description: "Container Logs & Debugging — When a container misbehaves, three commands cover almost every diagnosis: docker logs (what did it print), docker exec (poke…"
---

# 05 · Container Logs & Debugging

When a container misbehaves, three commands cover almost every diagnosis:
`docker logs` (what did it print), `docker exec` (poke at it while it's
running), and `docker inspect` (what is its actual configuration and
state).

## `docker logs`

```bash
docker logs myapp             # everything captured so far
docker logs -f myapp          # follow, like tail -f
docker logs --tail 100 myapp  # last 100 lines only
docker logs --since 10m myapp # only the last 10 minutes
docker logs -t myapp          # prefix each line with a timestamp
```

`docker logs` only shows what the container's PID 1 process wrote to
**stdout/stderr** — it is not a general-purpose log viewer for arbitrary
files the application might also write inside the container. An app that
logs only to `/var/log/app.log` and never to stdout will show nothing
here (fixable at the app's logging config, or by symlinking that file to
`/dev/stdout`).

## `docker exec` — a shell inside a running container

```bash
docker exec -it myapp sh          # interactive shell (bash if available)
docker exec myapp cat /etc/hosts  # one-off command, no interactive session
docker exec myapp env             # inspect the actual environment variables it sees
```

`-it` combines `-i` (keep stdin open) and `-t` (allocate a pseudo-TTY) —
both are needed for an interactive shell to behave like a normal
terminal; a one-off diagnostic command like `cat` needs neither.

## `docker inspect` for configuration and state

```bash
docker inspect myapp
docker inspect -f '{{.State.Status}}' myapp
docker inspect -f '{{json .Config.Env}}' myapp
docker inspect -f '{{.NetworkSettings.IPAddress}}' myapp
```

`--format`/`-f` with Go template syntax extracts one field instead of
scrolling through hundreds of lines of JSON — invaluable when scripting
health checks or debugging why a container isn't reachable on the network
you expect.

## Worked example: diagnosing a crash-looping container

```bash
docker ps -a --filter name=myapp
# STATUS shows "Restarting (1) 4 seconds ago" — crash-looping

docker logs --tail 50 myapp
# reveals the actual stack trace/error from the last crash

docker inspect -f '{{.State.ExitCode}}' myapp
# 1 — an application error, not a SIGKILL (which would show 137)

docker inspect -f '{{json .Config.Env}}' myapp | python3 -m json.tool
# confirms whether a required env var is actually set as expected
```

If the container restarts too fast to `exec` into it, temporarily
override its restart policy and command to keep it alive for inspection:

```bash
docker run -it --entrypoint sh myimage
# drops you into a shell instead of running the normal (crashing) CMD
```

## `docker stats` for resource-related symptoms

```bash
docker stats myapp --no-stream
```

Shows live CPU%, memory usage/limit, and network I/O — useful when the
symptom is "slow" or "OOM-killed" rather than "crashes immediately," and
complements `docker inspect`'s static config with a runtime snapshot.

## How It Actually Works

**Where `docker logs` output actually comes from.** The Docker daemon
doesn't read a container's stdout live off some shared pipe every time
you run `docker logs` — it captures stdout/stderr once, continuously, the
moment the container starts, via the configured **logging driver**
(default: `json-file`). Concretely, the container's stdout/stderr file
descriptors are connected to a pipe whose other end the daemon reads, and
the daemon writes each line to a JSON-lines file under
`/var/lib/docker/containers/<id>/<id>-json.log` (tagged with a
timestamp). `docker logs` is simply reading and formatting that file (or
tailing it, for `-f`) — which is also why logs survive a container
restart (same file, same container ID) but disappear entirely once you
`docker rm` it, and why an extremely chatty container with the default
driver can quietly fill your disk (module 07 covers alternative drivers).

**What `docker exec` actually attaches to, mechanically.** Unlike
`docker run`, which creates an entirely new PID/mount/network namespace
set, `docker exec` uses the `setns()` syscall to make a *new* process
join the **existing** namespaces of the target container's PID 1 — same
filesystem view (same mount namespace, so it sees the container's
overlayfs merged view, not the host's), same network namespace (same IP,
same open ports), same PID namespace (so `ps` inside the exec'd shell
shows the container's other processes, not the host's). This is why
`docker exec` feels like "getting a shell inside the container" despite
not being the original containerized process at all — it's a genuinely
separate process that happens to have joined all the same kernel
namespaces, which also means killing the `docker exec` shell has no
effect whatsoever on the container's actual PID 1 or its restart
behavior.

## Exercise

Run `docker run -d --name debug-demo alpine sh -c "echo starting; sleep 300"`.
Use `docker logs debug-demo` to see the startup message, `docker exec -it
debug-demo sh` to get a shell and confirm `ps` shows only that container's
processes (not your host's), then `docker inspect -f
'{{.State.Pid}}' debug-demo` to find its PID as seen from the host, and
compare it to the PID that process reports for itself inside the
container via `docker exec debug-demo sh -c "echo \$\$"`.
