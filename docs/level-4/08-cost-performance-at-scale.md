---
description: "Cost & Performance at Scale — At small scale, an oversized container or a bloated image costs nothing you'd notice. At fleet scale — dozens of services…"
---

# 08 · Cost & Performance at Scale

At small scale, an oversized container or a bloated image costs nothing
you'd notice. At fleet scale — dozens of services, hundreds of replicas,
billed by the CPU-second and the gigabyte — the same inefficiencies
compound into real invoices. This module covers the two levers that
actually move cost: **right-sizing resource requests** (module 03's
resource limits, revisited with a cost lens) and **image size** (module
04's build optimization, revisited the same way), plus how to measure
before guessing.

## Measure before you size

```bash
docker stats --no-stream
# CONTAINER   CPU %   MEM USAGE / LIMIT   MEM %   NET I/O
# web         3.20%   84MiB / 512MiB      16.41%  1.2MB / 800kB
```

A container using 84MiB against a 512MiB limit isn't "safely under
budget" — it's paying for 428MiB of capacity it never touches. Cloud
schedulers (Kubernetes, ECS, Swarm) generally bill or bin-pack against the
*request*, not the *actual usage*, so an inflated limit directly reduces
how many containers fit per host, which directly increases host count.

```bash
docker run -d --name web --memory=512m --memory-reservation=128m \
  --cpus=1.0 myapp:1.0

# after a week of docker stats sampling shows steady 90-110MiB usage:
docker update --memory=192m --memory-reservation=96m web
```

`--memory-reservation` (a soft limit) matters for bin-packing: the
scheduler can place more containers per host if it plans around typical
usage rather than the hard ceiling, while `--memory` (the hard limit)
still protects against a genuine leak by killing the container via OOM
rather than starving its neighbors.

## Right-sizing CPU: requests vs. bursts

```yaml
# compose.yml
services:
  api:
    image: myapp:1.0
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 256M
        reservations:
          cpus: "0.25"
          memory: 128M
```

The gap between `reservations` (guaranteed, used for scheduling/bin-packing)
and `limits` (the ceiling, used for throttling) is deliberate headroom for
bursts — a request-handling service that's idle 90% of the time and spikes
to 0.5 CPU briefly doesn't need a permanent 0.5-CPU reservation; it needs a
small reservation and a burst ceiling, which lets many such services
share a host's idle capacity instead of each reserving worst-case
capacity that mostly goes unused.

## Image size: the cost that's paid on every pull and every host

```dockerfile
# before: 1.1GB
FROM node:20
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
```

```dockerfile
# after: 178MB
FROM node:20-slim AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-slim
WORKDIR /app
COPY --from=build /app/node_modules ./node_modules
COPY . .
USER node
CMD ["node", "server.js"]
```

Image size is a cost multiplier in three places at once: registry storage
(billed per GB-month), network egress on every pull (billed per GB,
multiplied by however many hosts pull it), and node autoscaling latency
(a new node joining a cluster during a traffic spike can't serve requests
until it finishes pulling every image it needs to run — a 1.1GB image adds
real seconds-to-minutes to how fast you can scale out).

```bash
docker image ls myapp --format "{{.Size}}"
docker history myapp:1.0 --format "{{.Size}}\t{{.CreatedBy}}" | head -20
# find which single layer is responsible for the bulk of the size
```

## Worked example: sizing a real fleet from measured data

```bash
# collect real usage across every running replica for a week, not a guess
for c in $(docker ps --format '{{.Names}}' --filter name=api); do
  docker stats --no-stream --format "{{.Name}}: {{.MemUsage}} {{.CPUPerc}}" "$c"
done

# api_1: 96MiB / 512MiB  4.10%
# api_2: 103MiB / 512MiB 3.80%
# api_3: 91MiB / 512MiB  4.40%
```

All three replicas cluster around 100MiB and under 5% of one CPU, against
limits sized for a launch-day guess of 512MiB/1 CPU. Resizing to a
reservation just above the observed ceiling with headroom for growth:

```bash
docker service update \
  --limit-memory 200m --reserve-memory 128m \
  --limit-cpu 0.5 --reserve-cpu 0.15 \
  myapp_api
```

At 3 replicas this drops the *reserved* footprint from 1.5 CPU / 1.5GiB to
0.45 CPU / 384MiB — nearly a 3x reduction in what the scheduler must set
aside, meaning roughly 3x as many similarly-sized services can bin-pack
onto the same host fleet before it needs to scale out.

## How It Actually Works

**Why an oversized `--memory` limit costs money even when never hit.**
cgroups (the kernel mechanism behind `--memory`, covered in module 03)
enforce the limit as a ceiling on that specific container's cgroup — but
an orchestrator's scheduler makes placement decisions based on the *sum
of requested/reserved resources* on a host, not on real-time usage,
precisely because usage fluctuates and packing to a moving target risks
oversubscription. A host with 8GiB of memory and containers reserving
512MiB each can host 16 of them on paper regardless of whether they
actually use 100MiB or 500MiB — so the reservation number, not the
measured number, is what determines fleet density and therefore host
count and therefore bill.

**Why layer structure — not just final file count — drives pullable
image size.** A Docker image is a stack of read-only layers (module 04),
and `docker history` shows the size *added* by each instruction, not a
running total per file — deleting a file in a later `RUN` layer doesn't
shrink the image, because the earlier layer containing that file is still
part of the image's content-addressable layer stack and still gets pulled
in full. This is why the multi-stage pattern in the worked Dockerfile
matters mechanically: the `build` stage's layers (containing the full
`npm ci` cache and any dev dependencies) are never copied into the final
stage's layer stack at all — `COPY --from=build` copies specific
directories' *current contents* into a fresh layer in the final image,
leaving the bloated build-stage layers to exist only in the intermediate
image that gets discarded, not in anything that's ever pushed, pulled, or
billed for.

## 🔀 Related lessons on other tracks

- [ETL & Data Lake — 04 · Cost & Performance Optimization for Lake Storage](https://sigilipelli.github.io/etl-datalake-mastery-path/level-3/04-cost-performance-optimization/)
- [Pyspark — 03 · Cost Performance Tradeoffs](https://sigilipelli.github.io/pyspark-mastery-path/level-4/03-cost-performance-tradeoffs/)

## Exercise

Pick a service you've built in an earlier module, run it under load for a
few minutes while sampling `docker stats --no-stream` every 10 seconds,
and compute its actual peak CPU and memory. Set `--memory` and `--cpus` to
roughly 1.5x that peak (not a round guess), then re-run the same load and
confirm via `docker stats` that it never gets OOM-killed or CPU-throttled
at the new, smaller limits.
