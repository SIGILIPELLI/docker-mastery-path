---
description: "Production Image Hardening — Level 3's security module covered non-root users and minimal bases at an introductory level. Production hardening pushes…"
---

# 03 · Production Image Hardening

Level 3's security module covered non-root users and minimal bases at an
introductory level. Production hardening pushes further: distroless
bases with no shell at all, a fully read-only root filesystem, and
capability sets pared to exactly what the process needs.

## Distroless as the default assumption, not the exception

```dockerfile
FROM node:20 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=build /app/dist ./dist
COPY --from=build /app/node_modules ./node_modules
USER nonroot
EXPOSE 8000
CMD ["dist/server.js"]
```

A distroless image contains the language runtime and its direct
dependencies and essentially nothing else — no shell (`sh`, `bash`), no
package manager, no coreutils. This means `docker exec ... sh` simply
doesn't work against it, which is precisely the point: an attacker who
achieves code execution inside the container has no shell to pivot with,
no `curl`/`wget` to exfiltrate data or fetch a second-stage payload, and
no package manager to install tools. Debugging shifts to `docker logs`,
the app's own instrumentation, and — if truly necessary — a temporary
debug image built `FROM` the same base plus a shell, never deployed.

## A fully read-only root filesystem

```bash
docker run --read-only --tmpfs /tmp --tmpfs /run \
  -p 8000:8000 myapp:hardened
```

```yaml
services:
  api:
    image: myapp:hardened
    read_only: true
    tmpfs:
      - /tmp
      - /run
```

`--read-only` makes the entire root filesystem (everything outside
mounted volumes) immutable at the kernel level — a compromised process
cannot write a webshell, modify a binary, or plant a persistence
mechanism anywhere in the image's filesystem. Most apps still need *some*
writable space (temp files, a PID file, a Unix socket) — `--tmpfs`
mounts an in-memory writable filesystem at specific paths without
weakening the read-only guarantee everywhere else.

## Dropping every capability, then re-adding only what's proven necessary

```bash
docker run --read-only --tmpfs /tmp \
  --cap-drop=ALL \
  --security-opt=no-new-privileges \
  -p 8000:8000 myapp:hardened
```

`--security-opt=no-new-privileges` blocks a process (and anything it
`exec`s) from gaining additional privileges beyond what it started with —
specifically disabling the effect of setuid/setgid binaries and file
capabilities that would otherwise let a process escalate mid-execution,
closing a privilege-escalation path that surviving as non-root alone
doesn't.

## A fully hardened Dockerfile + run command together

```dockerfile
FROM node:20 AS build
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY . .

FROM gcr.io/distroless/nodejs20-debian12
WORKDIR /app
COPY --from=build --chown=nonroot:nonroot /app /app
USER nonroot
EXPOSE 8000
CMD ["src/server.js"]
```

```bash
docker run -d \
  --read-only --tmpfs /tmp \
  --cap-drop=ALL --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges \
  --memory=256m --cpus=1.0 \
  -p 8000:8000 myapp:hardened
```

This combines every mechanism covered so far, across three modules: no
root user, no shell in the image, read-only filesystem, minimal
capabilities, no privilege escalation, and resource limits (Level 3,
module 03).

## Worked example: confirming a hardened image resists a common attack

Suppose a dependency has a code-injection vulnerability that lets an
attacker get arbitrary shell commands executed inside the container.
Against a non-hardened image, this typically escalates to installing
tools, writing files, or reaching other hosts. Against the image above:
`sh -c "..."` simply fails (no shell binary present in distroless);
attempting to write anywhere outside `/tmp` fails (read-only root); any
attempt to `setuid` fails (`no-new-privileges`); attempting to open a raw
socket or bind a privileged port beyond the one explicitly re-added fails
(all capabilities dropped except `NET_BIND_SERVICE`). Each mechanism
independently narrows what a successful code-execution vulnerability can
actually accomplish.

## How It Actually Works

**Why `--read-only` plus `overlay2` interact the way they do.** Without
`--read-only`, a container's writable layer is the `UpperDir` in the
overlay2 stack from Level 3, module 06 — writes go there via copy-up.
`--read-only` doesn't remove that UpperDir's *existence*; it mounts the
final merged root filesystem view read-only at the VFS layer, so any
write syscall against a path not separately mounted writable (a
`--tmpfs`, a volume) is rejected by the kernel with `EROFS` before it
ever reaches the overlay filesystem's copy-up logic at all. This is a
strictly kernel-enforced VFS-level restriction, not an application
convention, which is why it holds even against a process that has
somehow bypassed the application's own logic — a raw `write()` syscall
against a read-only-mounted path fails regardless of what userspace code
issued it.

**Why `no-new-privileges` specifically targets setuid binaries and file
capabilities, not general privilege.** Historically, a process could
gain elevated privilege mid-execution in two main ways: executing a
setuid/setgid binary (which the kernel re-execs with the *file owner's*
UID/GID rather than the calling process's), or executing a binary with
Linux file capabilities attached (an extended attribute granting specific
capability bits to anyone who executes that file, regardless of their own
process's capability set). The `PR_SET_NO_NEW_PRIVS` prctl flag — what
`--security-opt=no-new-privileges` sets — tells the kernel to ignore both
mechanisms for this process and all its descendants for the rest of
their lifetime: `execve()` on a setuid binary or a file-capability binary
proceeds, but silently *without* granting the elevated privilege it would
otherwise confer. This closes a specific, well-known local
privilege-escalation vector that dropping capabilities from the
container's own process alone does not address, since dropped
capabilities only bound what the current process already has, not what a
setuid binary it might invoke could hand it back.

## Exercise

Take the hardened Dockerfile above, deliberately try to `docker exec -it
<container> sh` against it and observe the failure, then build a
throwaway debug variant (`FROM gcr.io/distroless/nodejs20-debian12:debug`,
which includes BusyBox) purely for local troubleshooting, and confirm it
is never referenced by the production stack file.
