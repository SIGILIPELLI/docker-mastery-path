# 07 · Zero-Downtime Deployments

Every deployment mechanism we've used so far — `docker service update`
(module 08, Level 3), `docker stack deploy`, a bare `docker run` restart —
answers the same underlying question differently: *how do you replace a
running container with a new one without a client ever seeing a
connection refused?* This module makes that question explicit and covers
the two ingredients that actually prevent downtime: **health gating**
(don't send traffic to a container until it's provably ready) and
**graceful shutdown** (don't kill a container while it's still serving a
request).

## Why a naive redeploy drops traffic

```bash
docker stop web && docker run -d --name web -p 8080:80 myapp:1.1
```

This has two separate failure windows:

1. Between `stop` and the new container binding port 8080, anything
   hitting the port gets connection-refused.
2. Even once the new container is listening, it may not be *ready* —
   still loading config, warming a cache, connecting to a database — so
   requests routed to it immediately can fail even though the port
   answers.

Zero-downtime deployment removes both windows by never taking capacity
away before its replacement is confirmed ready.

## Health checks as the readiness gate

```dockerfile
FROM node:20-slim
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY . .
HEALTHCHECK --interval=5s --timeout=3s --start-period=10s --retries=3 \
  CMD node healthcheck.js || exit 1
CMD ["node", "server.js"]
```

`healthcheck.js` should check more than "process is alive" — it should
verify the things that make the container *useful*, e.g. it can reach its
database connection pool:

```js
// healthcheck.js
const http = require('http');
const req = http.get('http://localhost:3000/healthz', (res) => {
  process.exit(res.statusCode === 200 ? 0 : 1);
});
req.on('error', () => process.exit(1));
req.setTimeout(2000, () => req.destroy());
```

`--start-period=10s` matters here: it gives the container a grace window
during which failed checks don't count against `--retries`, so a slow
cold start isn't mistaken for a crash. Without it, an orchestrator can
kill and endlessly restart a container that just needed a few more
seconds to boot.

## Rolling update with a health gate, in Swarm

```bash
docker service update \
  --image myapp:1.1 \
  --update-parallelism 1 \
  --update-delay 10s \
  --update-order start-first \
  --update-failure-action rollback \
  --health-cmd "node healthcheck.js" \
  --health-interval 5s \
  --health-retries 3 \
  web
```

`--update-order start-first` is the key flag for zero downtime: instead
of stopping the old task and then starting the new one (the default,
`stop-first`), Swarm **starts the new task alongside the old one**, waits
for it to pass its health check, and only then stops the old task. Combine
that with `--update-failure-action rollback` and a bad image never
displaces working capacity at all — Swarm reverts to the previous image
automatically if the new task fails its health check within the update
window.

## Graceful shutdown: the other half of the problem

Health gating protects incoming containers; graceful shutdown protects
outgoing ones. When Docker stops a container it sends `SIGTERM`, waits
`--stop-timeout` (default 10s), then sends `SIGKILL` if the process hasn't
exited. An application that ignores `SIGTERM` gets hard-killed mid-request
every time, regardless of how careful the deployment orchestration is.

```js
// server.js
const server = app.listen(3000);

process.on('SIGTERM', () => {
  console.log('SIGTERM received, draining connections...');
  server.close(() => {
    console.log('all connections drained, exiting');
    process.exit(0);
  });
  // safety valve: force-exit if drain takes too long
  setTimeout(() => process.exit(1), 8000);
});
```

`server.close()` stops accepting new connections but lets in-flight
requests finish before the callback fires — that's the difference between
a client getting a clean response and getting a reset connection during a
deploy.

## Worked example: a full rolling deploy with rollback

```yaml
# stack.yml
version: "3.9"
services:
  web:
    image: myapp:1.0
    deploy:
      replicas: 4
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
        failure_action: rollback
        monitor: 15s
      rollback_config:
        parallelism: 0
        order: stop-first
    healthcheck:
      test: ["CMD", "node", "healthcheck.js"]
      interval: 5s
      timeout: 3s
      retries: 3
      start_period: 10s
    ports:
      - "8080:80"
```

```bash
docker stack deploy -c stack.yml myapp
# ship a broken image on purpose to see the safety net work
docker service update --image myapp:broken myapp_web
docker service ps myapp_web
# watch: new tasks fail health checks, "monitor: 15s" window expires,
# failure_action: rollback reverts every replica back to myapp:1.0
# automatically -- no human had to notice the outage and intervene
```

`rollback_config.parallelism: 0` means "roll back all replicas at once"
rather than gradually — once you know the new image is bad, there's no
reason to phase the revert; get back to a known-good state immediately.

## How It Actually Works

**Why `start-first` genuinely avoids the connection-refused window.**
Under `stop-first` (Swarm's default), the manager removes a task's IP from
the service's IPVS load-balancing table *before* starting its
replacement, so for however long the new container takes to boot, the
service is running at N-1 capacity — and if that's the last replica on a
node, requests routed there hang or fail until the new task binds its
port. Under `start-first`, the new task is scheduled and its container
started **while the old task's IP is still in the IPVS table**; only after
the new task's health check reports healthy does the manager remove the
old task's entry and stop it. The two entries coexist in the kernel's IPVS
table for the overlap window, so IPVS keeps routing to the old,
known-working task the entire time the new one is initializing — capacity
never drops below N.

**Why SIGTERM-then-SIGKILL exists as a two-phase signal, mechanistically.**
`SIGTERM` is a *catchable* signal — the kernel delivers it to the
process's registered signal handler (if any) and lets userspace code
decide what to do, which is what makes `server.close()`-style draining
possible at all. `SIGKILL` is not catchable or blockable by design: the
kernel terminates the process at the scheduler level without ever handing
control back to userspace, guaranteeing a runaway or hung process
eventually dies even if its shutdown code itself has a bug. The
`--stop-timeout` window is Docker giving your `SIGTERM` handler a bounded
grace period before falling back to that unconditional guarantee — which
is exactly why the `setTimeout(() => process.exit(1), 8000)` safety valve
in the worked example matters: it makes the *application* exit cleanly
just under Docker's own kill deadline, rather than leaving the outcome to
whichever timeout fires first.

## Exercise

Take a Compose service with two replicas (`docker compose up --scale
web=2`) and add a health check that intentionally fails for the first 8
seconds after start (sleep before listening). Deploy an update with
`docker service update --update-order start-first` in a Swarm stack and
confirm via `docker service ps` that the old replica keeps running until
the new one's health check passes; then repeat with `--update-order
stop-first` and observe the capacity gap in between.
