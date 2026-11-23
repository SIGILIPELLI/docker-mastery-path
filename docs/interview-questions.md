---
title: "Docker Interview Questions and Answers (183 Questions)"
description: "183 Docker interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Docker Mastery Path."
---

# Docker Interview Questions and Answers

**183 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Docker Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [01 · What Is Docker &amp; Why Containers?](level-1/01-what-is-docker.md)

**Q1.** What is a container?

**Answer:** An isolated process (or group of processes) on the host's kernel, created from an image

The promise: 'it works on my machine' becomes 'it works everywhere the container runs'.

**Q2.** Which three Linux kernel features make containers possible?

**Answer:** Namespaces, cgroups and a layered (union) filesystem

Namespaces isolate the view; cgroups limit resources; layers share image data.

**Q3.** Why does a container start so much faster than a VM?

**Answer:** There's no second kernel to boot and no hardware to emulate

It starts about as fast as launching a normal process.

**Q4.** What is the trade-off in isolation strength?

**Answer:** Containers share the host kernel, so isolation is weaker than a VM's separate kernel

VMs offer very strong isolation at higher overhead.

### [02 · Installing Docker](level-1/02-installing-docker.md)

**Q5.** What do you install on macOS and Windows?

**Answer:** Docker Desktop, which bundles the daemon, CLI, a small Linux VM and a GUI (WSL2 backend on Windows)

Containers there run inside a Linux VM.

**Q6.** Why does Linux need no VM?

**Answer:** The host kernel is Linux, so Docker Engine runs natively

You install docker-ce, docker-ce-cli, containerd.io and the compose plugin.

**Q7.** Who can use the Docker socket by default, and how do you run docker without sudo?

**Answer:** root and members of the docker group; sudo usermod -aG docker $USER, then log in again

The lesson warns that docker group membership is effectively root-equivalent.

**Q8.** Why does the lesson say to cross-check docs.docker.com?

**Answer:** Installer screens and steps change over time

The lesson describes documented behaviour, not screenshots.

### [03 · Images vs Containers](level-1/03-images-vs-containers.md)

**Q9.** What is the relationship between an image and a container?

**Answer:** An image is a read-only template (like a class); a container is a running or stopped instance of it (like an object)

Many containers can be created from one image.

**Q10.** Which Dockerfile instructions produce new filesystem layers?

**Answer:** RUN, COPY and ADD, which change the filesystem

Each layer is immutable and identified by a content hash.

**Q11.** Where do a container's runtime file changes go?

**Answer:** A thin writable layer unique to that container; the image layers are never touched

So containers from the same image don't interfere with each other.

**Q12.** Two images are both built FROM python:3.12-slim. What does Docker store?

**Answer:** The shared base layer once on disk

Layers are shared even across images.

### [04 · Running Your First Container](level-1/04-first-container.md)

**Q13.** What does `docker run hello-world` do, in order?

**Answer:** Resolve hello-world:latest, check the local cache, pull if missing, create the container, start its CMD/ENTRYPOINT as PID 1, attach

It explains most 'why did it pull from the internet' questions.

**Q14.** What do -i and -t mean in `docker run -it ubuntu bash`?

**Answer:** -i keeps STDIN open; -t allocates a pseudo-TTY

Together they behave like a normal terminal.

**Q15.** What is the container's main process?

**Answer:** The image's configured CMD or ENTRYPOINT, running as PID 1 in the container's PID namespace

When it exits, the container stops.

**Q16.** What does Docker use by default for the registry?

**Answer:** Docker Hub

The default tag is latest.

### [05 · The Dockerfile](level-1/05-the-dockerfile.md)

**Q17.** Why prefer WORKDIR /app over `RUN cd /app && ...`?

**Answer:** cd in a RUN affects only that instruction's shell, not later instructions; WORKDIR persists and creates the directory

WORKDIR sets the directory for later RUN, CMD, COPY and ADD.

**Q18.** What does EXPOSE 8000 do?

**Answer:** Documents the port (metadata); it does not publish it. -p publishes, and -P publishes all exposed ports to random host ports

It's a hint for humans and tools.

**Q19.** Why prefer COPY over ADD for plain files?

**Answer:** ADD has surprising extras (auto-extracting tarballs, fetching URLs)

COPY copies from the build context.

**Q20.** How do RUN and CMD differ?

**Answer:** RUN executes at build time and commits a layer; CMD only runs when a container starts

Use the exec (JSON array) form for CMD.

### [06 · Building Images](level-1/06-building-images.md)

**Q21.** What is the build context in `docker build -t hello-app:1.0 .`?

**Answer:** The directory (.) whose contents are sent to the daemon; COPY can't reference files outside it

Use -f for a differently named Dockerfile.

**Q22.** What does the daemon do for each instruction during a build?

**Answer:** Checks for a cached layer for that instruction plus its parent; reuses it or executes in a temporary container and commits a new layer

Caching is explained in Level 2.

**Q23.** What does -t hello-app:1.0 do?

**Answer:** Tags the image with repository hello-app and tag 1.0 (you can pass -t multiple times)

The default tag is latest.

**Q24.** What does .dockerignore do?

**Answer:** Excludes files from the build context, like .gitignore does for git

It keeps the context small and secrets out.

### [07 · Container Lifecycle](level-1/07-container-lifecycle.md)

**Q25.** What is `docker run` shorthand for?

**Answer:** docker create followed by docker start

That's why `docker run --name x` fails the second time: the container already exists.

**Q26.** What's the difference between docker stop and docker kill?

**Answer:** stop sends SIGTERM, waits, then SIGKILL; kill sends SIGKILL immediately

stop gives the app a chance to shut down cleanly.

**Q27.** What does docker pause do?

**Answer:** Freezes process execution with the cgroup freezer while retaining memory/state

unpause resumes it.

**Q28.** What does `docker rm -f` do?

**Answer:** Force: kills a running container and removes it in one step

Without -f the container must be stopped first.

### [08 · Volumes &amp; Bind Mounts](level-1/08-volumes-bind-mounts.md)

**Q29.** What happens to data in a container's writable layer when it's removed?

**Answer:** It's destroyed with the container

db2 would start with a fresh empty database.

**Q30.** What is a volume?

**Answer:** Storage managed by Docker under its own host location, referenced by name; its lifetime is independent of any container

For example -v pgdata:/var/lib/postgresql/data.

**Q31.** What happens if the named volume doesn't exist when you use -v pgdata:/path?

**Answer:** Docker creates it automatically on first use

docker volume create just makes intent clearer.

**Q32.** What is a bind mount?

**Answer:** A specific path on the host mapped directly into the container

It's handy for development; volumes suit durable app data.

### [09 · Networking Basics](level-1/09-networking-basics.md)

**Q33.** What does the default bridge network provide?

**Answer:** A private virtual switch: each container gets its own interface and private IP, outbound traffic is NATed, and nothing outside can reach it unless ports are published

The host side commonly shows as docker0.

**Q34.** What does `docker run -d -p 8080:80 nginx` do?

**Answer:** Creates a NAT rule forwarding host port 8080 to container port 80

-p is &lt;host&gt;:&lt;container&gt;.

**Q35.** Why create a user-defined bridge network?

**Answer:** Containers get DNS-based discovery by name (docker's embedded DNS), unlike the default bridge where only IPs work

From frontend, the hostname backend resolves automatically.

**Q36.** Why are IP addresses fragile for container-to-container communication?

**Answer:** They can change on restart

Use names on a user-defined network.

### [10 · Docker Hub &amp; Registries](level-1/10-docker-hub-registries.md)

**Q37.** What does `docker pull nginx:1.27` resolve to?

**Answer:** docker.io/library/nginx:1.27, where Docker Hub and the library/ namespace are implicit defaults

It then downloads the manifest and any missing layers in parallel.

**Q38.** What is an official image?

**Answer:** An unprefixed image like nginx, maintained and reviewed through Docker Hub, with docs and regular security patches

Good default base images.

**Q39.** How must you name an image to push it to Docker Hub?

**Answer:** &lt;your-username&gt;/&lt;repo&gt;[:tag], using docker tag to retag without rebuilding

For example docker tag hello-app:1.0 yourusername/hello-app:1.0.

**Q40.** Where does `docker login` store credentials by default?

**Answer:** ~/.docker/config.json, ideally backed by an OS credential store rather than plaintext

Subsequent push and pull use it.

### [11 · Project — Containerize a Small App](level-1/11-project-containerize-app.md)

**Q41.** Why does the Dockerfile copy requirements.txt and install before copying app.py?

**Answer:** Editing the app later won't invalidate the slower dependency-install layer

This is the layer-caching order from modules 05 and 06.

**Q42.** What does -v visit-counter-data:/data achieve?

**Answer:** count.txt lives in a named volume, so the count outlives the container

The count survives removing and re-creating the container.

**Q43.** What does --restart unless-stopped do?

**Answer:** Brings the container back after a crash or host reboot, unless you stopped it

It's a lifecycle policy from module 7.

**Q44.** What does -p 8080:5000 mean?

**Answer:** Host port 8080 forwards to the container's Flask port 5000

The app listens on 0.0.0.0:5000 inside.

## Level 2 · Intermediate

### [01 · Multi-Stage Builds](level-2/01-multi-stage-builds.md)

**Q45.** What problem does a multi-stage build solve?

**Answer:** Build tooling (compilers, dev dependencies, sources) never ends up in the shipped image; only artifacts you COPY --from= survive

The golang:1.22 image is over 800 MB, yet the running container needs only one static binary.

**Q46.** How do you move an artifact from the build stage to the final stage?

**Answer:** COPY --from=&lt;stage&gt;

Everything else in the build stage is discarded.

**Q47.** What does a multi-stage Dockerfile contain?

**Answer:** More than one FROM instruction

Each FROM starts a new stage.

**Q48.** What does a naive Dockerfile ship to production for a Go app?

**Answer:** The whole toolchain, standard library sources and build cache

That's what multi-stage avoids.

### [02 · Docker Compose Basics](level-2/02-docker-compose-basics.md)

**Q49.** What does Compose let you do?

**Answer:** Describe a multi-container app in YAML and bring it up or down with one command

It replaces long docker run, network create and volume create chains.

**Q50.** What does depends_on control?

**Answer:** Start order only; it doesn't wait for the dependency to be ready

Readiness needs healthchecks (module 06).

**Q51.** What does `docker compose down` remove?

**Answer:** Containers and the network, but it keeps named volumes

Add -v to remove volumes.

**Q52.** Why use `docker compose up -d --build`?

**Answer:** To force a rebuild of the image from its Dockerfile, since plain up reuses an existing image

Needed after code or Dockerfile changes.

### [03 · Environment Variables &amp; .env Files](level-2/03-environment-variables-env-files.md)

**Q53.** What is special about a file named .env in a Compose project directory?

**Answer:** It's loaded automatically for variable substitution inside docker-compose.yml itself, not injected into containers

Use environment: ${VAR} to pass values into the container.

**Q54.** How do ENV, environment:/-e, and .env differ?

**Answer:** ENV is baked into the image; environment/-e is a runtime override; .env substitutes values into the compose file text

A common mistake is expecting .env to inject variables into the container.

**Q55.** What does `docker compose config` do?

**Answer:** Renders the final YAML with substitutions applied so you can verify what's sent to the Engine

Use it before trusting a nested ${VAR}.

**Q56.** Why avoid putting secrets in .env files?

**Answer:** They tend to get committed by accident; use Docker secrets or a secret manager

One compose file can serve several environments with differing values only.

### [04 · Image Layering &amp; Build Cache](level-2/04-image-layering-build-cache.md)

**Q57.** What happens when one layer misses the cache?

**Answer:** Every subsequent layer misses too, because their starting point changed

Order instructions from least to most frequently changing.

**Q58.** What is a COPY layer's cache key?

**Answer:** The instruction plus a checksum of the copied files, so one changed byte invalidates it and everything after

For RUN, it's the exact command string plus the parent layer.

**Q59.** Why put COPY . . after the pip install layer?

**Answer:** Edits to app code won't invalidate the slow dependency-install layer

Measured: the pip layer says CACHED and the build takes about 1 s.

**Q60.** What does a BuildKit cache mount (--mount=type=cache) do?

**Answer:** Persists a directory like pip's download cache across builds, independent of the layer cache

Useful when requirements.txt changes.

### [05 · Container Logs &amp; Debugging](level-2/05-logs-and-debugging.md)

**Q61.** What does `docker logs` show?

**Answer:** What the container's PID 1 process wrote to stdout/stderr

An app that logs only to /var/log/app.log shows nothing; symlink it to /dev/stdout.

**Q62.** When do you need -it with docker exec?

**Answer:** For an interactive shell; a one-off command like cat needs neither

-i keeps stdin open; -t allocates a pseudo-TTY.

**Q63.** What does `docker inspect -f '{{.State.Status}}' myapp` show?

**Answer:** A single field of the container's state, using a Go template

docker inspect shows full configuration and state.

**Q64.** How do you follow logs and show only the last 100 lines?

**Answer:** docker logs -f --tail 100 myapp

--since 10m limits by time; -t adds timestamps.

### [06 · Health Checks](level-2/06-health-checks.md)

**Q65.** What does 'Up' in docker ps tell you?

**Answer:** Only that the main process hasn't exited, not that the app can serve traffic

A health check gives a real readiness signal: healthy, unhealthy or starting.

**Q66.** What does --start-period do?

**Answer:** A grace period after start during which failures don't count toward --retries

It suits slow-starting apps.

**Q67.** What exit codes does a health check command use?

**Answer:** 0 for healthy and 1 for unhealthy

curl -f fails on an HTTP error status.

**Q68.** What does `depends_on: api: condition: service_healthy` add?

**Answer:** The worker waits until the api's health check is healthy, not merely running

Without a health check, depends_on only guarantees the container process started.

### [07 · Compose Networks &amp; Volumes](level-2/07-compose-networks-volumes.md)

**Q69.** Why declare explicit networks?

**Answer:** By default every service shares one network, so a compromised web container could reach services it shouldn't; tiers limit that

A frontend-only proxy would have no route to the db.

**Q70.** A web service is on frontend and backend; db is on backend only. Who can reach db?

**Answer:** web (shared backend network), but not a service on frontend alone

Services need a shared network to talk.

**Q71.** What does `external: true` on a volume mean?

**Answer:** The volume already exists and Compose must not create or manage its lifecycle; `down -v` won't remove it

Useful for production data that must outlive any project's teardown.

**Q72.** What is the difference between a named volume and a bind mount in Compose?

**Answer:** A named volume is Docker-managed and independent of containers; a bind mount maps a literal host path and ties the container to that host's layout

Bind mounts suit init scripts and live-reload source.

### [08 · Managing Images &amp; Tags](level-2/08-managing-images-tags.md)

**Q73.** What does `docker tag` do?

**Answer:** Adds another name pointing at the same image ID; nothing is copied

Like a hard link adds a name for the same inode.

**Q74.** Why is :latest risky for deployments?

**Answer:** It's mutable: it silently points at whatever was last pushed

Pin deploys to an immutable tag such as a git SHA.

**Q75.** What is a dangling image?

**Answer:** An image with no tag (&lt;none&gt;:&lt;none&gt;), usually the old image after a tag moved to a rebuild

docker image prune removes them.

**Q76.** How does `docker image prune -a` differ from plain prune?

**Answer:** It removes all images not used by a running container, including tagged ones: fine for CI agents, dangerous on a workstation

It's much more aggressive.

### [09 · Compose for Multi-Service Apps](level-2/09-compose-multi-service-apps.md)

**Q77.** How does the web service reach the api in the three-tier stack?

**Answer:** By its service name via Compose's default network DNS: API_URL=http://api:8000

The same applies to db:5432 and cache:6379.

**Q78.** Why does the api use depends_on db with condition: service_healthy?

**Answer:** So it starts only after the database is actually ready

The api itself has a health check too.

**Q79.** What does a realistic app stack usually include?

**Answer:** A web tier, an API, a database and often a cache, wired with start order, environment and networks

The module combines modules 02–08.

**Q80.** Which service publishes a host port in the example?

**Answer:** web, with 3000:3000, while api and db are internal

Internal services needn't publish ports.

### [10 · Project — Multi-Container Compose App](level-2/10-project-compose-app.md)

**Q81.** Which networks keep the data tier private in the link-shortener stack?

**Answer:** data is internal: true, so db and cache aren't reachable from outside; only api joins public and data, and proxy joins public only

The proxy can't reach db or cache directly (no shared network).

**Q82.** How does the api's multi-stage Dockerfile keep the runtime image small?

**Answer:** An npm ci stage installs dependencies; the node:20-slim runtime copies only node_modules with COPY --from=deps

The runtime stage also sets NODE_ENV=production and a HEALTHCHECK.

**Q83.** What does the db health check pg_isready -U app enable?

**Answer:** depends_on condition service_healthy, so the api starts only once Postgres is ready

The cache only needs service_started.

**Q84.** How is configuration varied between environments?

**Answer:** A .env file supplies versions and the port via ${VAR} substitution in the compose file

For example image: nginx:${NGINX_VERSION} and ${APP_PORT}:80.

## Level 3 · Advanced

### [01 · Docker Networking Deep Dive](level-3/01-networking-deep-dive.md)

**Q85.** Which network driver gives no isolation and removes the need for -p mapping?

**Answer:** host: the container shares the host's network namespace

Two containers can't both bind the same port; rarely appropriate as a default.

**Q86.** What is the practical reason to use a user-defined bridge instead of the default docker0 bridge?

**Answer:** It provides embedded DNS, so containers resolve each other by name; the default bridge only supports IPs and legacy --link

Compose creates user-defined networks automatically.

**Q87.** What is the overlay driver for?

**Answer:** Multi-host networking, such as Swarm services whose containers run on different machines

It's created with --driver overlay --attachable.

**Q88.** When would you use `none` or `macvlan`?

**Answer:** none for fully isolated jobs with no networking; macvlan to give a container its own MAC/IP on the physical LAN

They're niche drivers.

### [02 · Security Basics](level-3/02-security-basics.md)

**Q89.** Why avoid running as root inside the container?

**Answer:** Without user-namespace remapping, root in the container maps to the host's root UID

Most web apps and workers shouldn't need it; use USER appuser or the image's node user.

**Q90.** What does `COPY --chown=node:node . .` avoid?

**Answer:** A separate RUN chown -R layer that would duplicate the copied files' size in the layer diff

Ownership is set at copy time.

**Q91.** What is the trade-off of distroless images?

**Answer:** Smallest attack surface (no shell or package manager), but you can't docker exec into them with a shell for debugging

alpine is tiny but uses musl, which can cause compatibility issues. Choose by runtime needs, not size alone.

**Q92.** What does `docker scout cves myapp:latest` do?

**Answer:** Checks installed packages against vulnerability databases and reports CVEs by severity, including which layer introduced them

Scan before shipping.

### [03 · Resource Limits (CPU/Memory)](level-3/03-resource-limits.md)

**Q93.** What happens when a container exceeds --memory?

**Answer:** The kernel OOM killer terminates it (exit code 137); docker inspect OOMKilled shows true

A container that dies with 137 and no app error is a strong hint.

**Q94.** What does setting --memory-swap equal to --memory do?

**Answer:** Disables swap for the container

--memory-swap caps RAM plus swap combined.

**Q95.** How do --cpus and --cpu-shares differ?

**Answer:** --cpus is an absolute cap; --cpu-shares is a relative weight that only matters under contention

With an idle host a low-share container can still use 100% of a core.

**Q96.** What does --memory-reservation do?

**Answer:** A soft limit enforced only under host memory pressure

--cpuset-cpus pins to specific cores.

### [04 · Multi-Container App Patterns](level-3/04-multi-container-patterns.md)

**Q97.** What is the sidecar pattern?

**Answer:** A container running alongside the main one (sharing network and often a volume) adds a capability like log shipping without changing the main image

In the example app writes logs to a shared volume and log-shipper reads them read-only.

**Q98.** What does an ambassador container do?

**Answer:** A small proxy between the app and an external dependency, so the app talks to a stable local address while the real endpoint can change

Only haproxy.cfg changes if the database's address changes.

**Q99.** What is an init container pattern, and what condition makes the api wait for it?

**Answer:** A job that runs to completion before the main container (e.g. migrations); depends_on condition service_completed_successfully

The migrate service uses restart: "no".

**Q100.** Why did the log-shipper mount applogs read-only (:ro)?

**Answer:** It only needs to read the logs, which prevents it from altering them

The app's image never needed to know fluent-bit exists.

### [05 · Private Registries](level-3/05-private-registries.md)

**Q101.** How does Docker know where to push localhost:5000/myapp:1.0?

**Answer:** The registry hostname and port are part of the image name

Without a prefix, the implicit registry is docker.io/library.

**Q102.** Why does Docker refuse plain-HTTP registries other than localhost?

**Answer:** It treats them as insecure; use TLS (a proxy or the registry's own), or an insecure-registries entry for local testing only

insecure-registries is a host-wide opt-out of transport security.

**Q103.** How do you add basic authentication to the registry:2 image?

**Answer:** An htpasswd file mounted at /auth with REGISTRY_AUTH=htpasswd (and realm and path variables), then docker login

An unauthenticated registry is for local experimentation only.

**Q104.** Name a managed registry option.

**Answer:** ECR, GCR, Azure ACR, GitHub Container Registry or Harbor

For ECR: aws ecr get-login-password | docker login --password-stdin.

### [06 · Storage Drivers &amp; Volume Plugins](level-3/06-storage-drivers-volume-plugins.md)

**Q105.** Which storage driver is the modern Linux default?

**Answer:** overlay2

Check with docker info | grep "Storage Driver".

**Q106.** In overlay2, what are LowerDir, UpperDir and MergedDir?

**Answer:** LowerDir: read-only image layers; UpperDir: the container's single writable directory; MergedDir: the unified root filesystem the process sees

Everything the container writes lands in UpperDir.

**Q107.** What does a volume plugin enable?

**Answer:** Backing volumes with network or cloud storage, e.g. a volume that survives the host being terminated or is shared across hosts

From inside the container /data looks like a normal directory.

**Q108.** How can a volume use NFS without a plugin?

**Answer:** The built-in local driver with --opt type=nfs and --opt o=addr=...,rw and device=:/path

The lesson shows docker volume create --driver local.

### [07 · Logging Drivers &amp; Centralized Logging](level-3/07-logging-drivers.md)

**Q109.** Which driver was used by default in the earlier logs lesson?

**Answer:** json-file

Logs sit on the host and vanish when the container is removed.

**Q110.** Why send logs to a central system in production?

**Answer:** With many hosts and short-lived containers, logs must leave the host before the container and its log file disappear

For example --log-driver=syslog --log-opt syslog-address=udp://...

**Q111.** How do you set a logging driver for one container?

**Answer:** --log-driver on docker run, with --log-opt options

It can also be set in daemon.json for all containers.

**Q112.** What happens to `docker logs` when a non-json-file driver is used?

**Answer:** It may not work for drivers that don't keep a local copy, so check the driver's capabilities

Some drivers only ship logs away.

### [08 · Docker Swarm Basics](level-3/08-docker-swarm-basics.md)

**Q113.** What does `docker swarm init --advertise-addr ...` do?

**Answer:** Turns the host into a Swarm manager and prints a join command with a token for workers

Workers join with docker swarm join --token ... :2377.

**Q114.** What is a Swarm service?

**Answer:** A desired state (image, replicas, ports) that the manager continuously reconciles the cluster toward

If a node dies, a replacement replica is scheduled on a healthy node.

**Q115.** What does `--update-parallelism 1 --update-delay 10s` do?

**Answer:** A rolling update: replace one replica, wait 10 s for it to become healthy, then continue

It avoids a total outage if the new image is broken.

**Q116.** How does `docker stack deploy -c stack.yml myapp` relate to Compose?

**Answer:** Swarm reuses Compose file syntax (with deploy: keys) for multi-service deployments

Overlay networks are needed for multi-host stacks.

### [09 · Secrets &amp; Config Management](level-3/09-secrets-config-management.md)

**Q117.** Why are plain environment variables a poor place for secrets?

**Answer:** They're visible via docker inspect to anyone with Engine API access, and appear in Swarm's Raft state and process listings on some platforms

Secrets are mounted as files instead.

**Q118.** Where is a Docker secret mounted inside the container?

**Answer:** /run/secrets/&lt;name&gt;, an in-memory tmpfs file never written to the writable layer

POSTGRES_PASSWORD_FILE tells the image to read the value from that file.

**Q119.** How do configs differ from secrets?

**Answer:** Configs are mounted similarly but are for non-sensitive files (nginx.conf, feature flags) and lack the same encryption-at-rest guarantees

Both are managed, versioned objects.

**Q120.** Why must a stack file mark a secret as external: true?

**Answer:** Swarm deliberately doesn't allow raw secret values inline, since stack files end up committed to version control

Create it beforehand with docker secret create.

### [10 · Project — Secure, Resource-Limited Production Stack](level-3/10-project-secure-stack.md)

**Q121.** Which hardening steps does the API Dockerfile use?

**Answer:** Multi-stage build with npm ci --omit=dev, COPY --chown=node:node, USER node, NODE_ENV=production and a HEALTHCHECK

The runtime image contains no dev tooling.

**Q122.** How is the database password provided to the API?

**Answer:** Via a Docker secret mounted as a file, referenced by DATABASE_URL_FILE=/run/secrets/database_url

The secrets file is gitignored.

**Q123.** What do resource limits and reservations in the stack do?

**Answer:** limits cap CPU and memory per replica; reservations guarantee a minimum

For example the api: limits 1.0 CPU/256M, reservations 0.25 CPU/128M.

**Q124.** What does the log-shipping sidecar add?

**Answer:** Centralised logs without changing the app image

It's one of the multi-container patterns from module 4.

## Level 4 · Master

### [01 · Docker in CI/CD Pipelines](level-4/01-docker-in-cicd.md)

**Q125.** Why tag images with the commit SHA in CI?

**Answer:** Every pushed image is traceable to the exact source that produced it, essential when debugging production weeks later

It uses the tagging strategy from Level 2.

**Q126.** What is the benefit of a dedicated `test` stage in the Dockerfile?

**Answer:** CI fails the build on test failure before producing the artifact, and tests run against the same dependency state as production, with the stage never shipping

Build with docker build --target test.

**Q127.** How should registry credentials be supplied in the workflow?

**Answer:** From repository secrets (${{ secrets.REGISTRY_TOKEN }}), never in the file

docker/login-action uses them.

**Q128.** What does the pipeline do in order?

**Answer:** Checkout, set up Buildx, log in, build and test, build the production image, push

Triggered on push to main.

### [02 · Orchestration Handoff (Kubernetes/Swarm)](level-4/02-orchestration-handoff.md)

**Q129.** Which signals suggest moving beyond Compose/Swarm to Kubernetes?

**Answer:** Many hosts with varied hardware, multiple teams needing namespaces and RBAC, metric-based autoscaling, and a need for the ecosystem (Helm, operators, meshes)

The lesson is honest that not every project reaches that point.

**Q130.** When is Swarm or plain Compose still the right size?

**Answer:** A single team, a handful of hosts and predictable load, where Swarm's simplicity is an advantage

Adopting Kubernetes for its reputation alone adds burden without benefit.

**Q131.** What do Swarm's service and task map to in Kubernetes?

**Answer:** Deployment + Service, and Pod (which can hold several co-scheduled containers sharing a network namespace)

Docker secret maps to Secret and docker config to ConfigMap.

**Q132.** How does Kubernetes handle networking compared to Swarm's overlay?

**Answer:** It delegates to a pluggable CNI network plugin

That's a key architectural difference.

### [03 · Production Image Hardening](level-4/03-production-image-hardening.md)

**Q133.** Why is a distroless image safer?

**Answer:** No shell, package manager or coreutils: an attacker has no shell to pivot with and no curl/wget to fetch payloads

Debugging shifts to logs, app instrumentation or a temporary debug image.

**Q134.** What does --read-only do, and why add --tmpfs?

**Answer:** Makes the root filesystem immutable (no webshell or persistence), while --tmpfs gives in-memory writable paths such as /tmp

The read-only guarantee stays everywhere else.

**Q135.** What does --security-opt=no-new-privileges do?

**Answer:** Blocks a process and its children from gaining extra privileges via setuid/setgid binaries or file capabilities

It pairs with --cap-drop=ALL, re-adding only what's proven necessary.

**Q136.** What does --cap-drop=ALL do?

**Answer:** Removes every Linux capability from the container's processes; add back only what's needed

Least privilege.

### [04 · Build Optimization at Scale](level-4/04-build-optimization-at-scale.md)

**Q137.** Why use per-branch cache tags with a fallback chain?

**Answer:** A single cache tag becomes a write-contention bottleneck; each branch reads its own cache then falls back to main's

New branches get fast first builds from main's layers.

**Q138.** What does cache-to ... mode=max do?

**Answer:** Exports layers from all stages, not just the final image, for better cache reuse

It's used with type=registry.

**Q139.** What do remote builders provide?

**Answer:** Consistent, powerful build hosts used via docker buildx create --driver remote

Builds run on a build farm rather than on each runner.

**Q140.** What shift does this lesson recommend?

**Answer:** Treat build infrastructure as a system to optimise, not just individual Dockerfiles

Many services and concurrent jobs make it a system problem.

### [05 · Designing a Containerized Architecture](level-4/05-containerized-architecture-design.md)

**Q141.** What two questions does 'one process per container' really stand for?

**Answer:** Does it need to scale independently, and does it change or deploy independently?

A CPU-bound worker and a light API gateway have different demand curves.

**Q142.** When may components legitimately stay in one container?

**Answer:** When they always scale together, always deploy together and are maintained by the same team

Over-decomposing adds coordination and network-hop overhead without benefit.

**Q143.** What is the downside of a direct HTTP/gRPC call between services versus a queue?

**Answer:** It couples the caller to the callee's availability in real time; a queue decouples uptime at the cost of eventual consistency

Decide synchronous or asynchronous at each boundary.

**Q144.** Why prefer each service owning its data store?

**Answer:** Two services sharing tables aren't really separate; schema changes in one can silently break the other

Expose data to others only through the service's own API.

### [06 · Observability for Containers](level-4/06-observability-for-containers.md)

**Q145.** How do metrics typically reach a collector?

**Answer:** The app exposes a /metrics HTTP endpoint (Prometheus format) that the collector scrapes periodically

Prometheus reaches api:8000 via Compose service-name DNS.

**Q146.** What does cAdvisor give you?

**Answer:** Per-container CPU, memory, network and disk metrics from cgroups, without instrumenting the app

It reads the same cgroup data as docker stats.

**Q147.** What does distributed tracing need to cross container boundaries?

**Answer:** A trace ID propagated on each hop (HTTP headers or message metadata) so the path and per-hop latency can be reconstructed

Services send spans to an OpenTelemetry collector.

**Q148.** Which three pillars of observability are covered across the course?

**Answer:** Logs, metrics and traces

Logs came in Level 3, module 07.

### [07 · Zero-Downtime Deployments](level-4/07-zero-downtime-deployments.md)

**Q149.** What are the two failure windows of `docker stop web && docker run ...`?

**Answer:** Connection refused between stop and the new container binding the port, and the new container being listening but not ready

Zero downtime never removes capacity before its replacement is ready.

**Q150.** What are the two ingredients that prevent downtime?

**Answer:** Health gating (no traffic until provably ready) and graceful shutdown (don't kill a container mid-request)

The health check should verify the container is useful, e.g. can reach its database.

**Q151.** Why does --start-period=10s matter?

**Answer:** Failed checks during a slow cold start don't count against --retries, so it isn't mistaken for a crash and restarted endlessly

Otherwise an orchestrator can kill a container that just needed more seconds.

**Q152.** What do --update-order start-first and --update-failure-action rollback do?

**Answer:** Start the new replica before stopping the old one, and roll back automatically if the update fails

They pair with a health command.

### [08 · Cost &amp; Performance at Scale](level-4/08-cost-performance-at-scale.md)

**Q153.** Why isn't 84MiB used against a 512MiB limit 'safely under budget'?

**Answer:** You're paying for 428MiB never touched, since schedulers bill and bin-pack against the request, not actual usage

Inflated limits reduce containers per host and raise host count.

**Q154.** What should you do before right-sizing?

**Answer:** Measure with docker stats over time (e.g. a week), then adjust, as in docker update --memory=192m

Steady usage was 90–110 MiB in the example.

**Q155.** Why does --memory-reservation matter for bin-packing?

**Answer:** The scheduler can plan around typical usage while --memory still protects against leaks via OOM kill

It is the soft limit.

**Q156.** Which two levers move cost most?

**Answer:** Right-sizing resource requests and image size

Both compound at fleet scale.

### [09 · Disaster Recovery &amp; Backups](level-4/09-disaster-recovery-backups.md)

**Q157.** What does NOT need backing up in a containerised system?

**Answer:** Containers and images themselves, which are rebuilt from source and redeployed from the stack file

Data in volumes and non-reproducible registry images do need backups.

**Q158.** How does the sidecar backup pattern work?

**Answer:** A throwaway alpine container mounts the volume read-only plus a host directory, and tar writes a timestamped archive

The read-only mount means the backup can't corrupt live data.

**Q159.** Why restore into a new volume name first?

**Answer:** You can verify the restored data before replacing the live volume

A backup that's never been restored is a hope, not a plan.

**Q160.** Why are registry images at risk even though they're 'reproducible'?

**Answer:** A base image tag can be repointed upstream or the build context may disappear, so an old build might not be reconstructible

Pin by digest and back up registry contents.

### [10 · Capstone Project](level-4/10-capstone-project.md)

**Q161.** Which Dockerfile baseline does the capstone treat as non-optional for production?

**Answer:** Non-root USER, a multi-stage build without dev dependencies, and a real health check

They come from modules 03, 04 and 07.

**Q162.** Which services make up the capstone system?

**Answer:** A stateless web API, a Postgres db and Redis cache on named volumes, and a reverse proxy for TLS and routing

web is replicated with limits and rolling updates.

**Q163.** What does the web service's update_config provide?

**Answer:** Rolling updates one at a time with start-first order and automatic rollback on failure

parallelism 1, delay 10s, order start-first, failure_action rollback.

**Q164.** What is the point of the capstone's written deliverable?

**Answer:** Putting everything together correctly and writing down why, like a production handoff document

There's no new syntax in it.

## Mixed questions across all levels

**Q165.** A Dockerfile has these lines. After editing only app.py, which steps rebuild (cache miss)?

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

**Answer:** COPY . . and then RUN pip install (every later layer misses once one layer misses)

COPY . . has a changed checksum, so it and every layer after it rebuild. Verified with a small cache simulation. Copying requirements.txt first avoids the reinstall.

**Q166.** After reordering to `COPY requirements.txt .` → `RUN pip install` → `COPY . .`, you edit only app.py. What rebuilds?

**Answer:** Only COPY . .; the pip install layer is CACHED

The dependency manifest didn't change, so the install layer is reused. Verified by simulation.

**Q167.** `docker run --name web nginx` worked yesterday and fails today with a name conflict. Why, and what fixes it?

**Answer:** docker run is create + start, so the container web still exists (Exited); docker rm web, or docker start web

A stopped container keeps its name until removed.

**Q168.** A container exits with code 137 and no error in its logs. What should you check first?

**Answer:** Whether the kernel OOM killer stopped it: docker inspect -f '{{.State.OOMKilled}}'

Exceeding --memory gets the process killed rather than throttled.

**Q169.** `EXPOSE 8000` is in the Dockerfile, but `curl localhost:8000` from the host fails. Why?

**Answer:** EXPOSE is only documentation; you must publish with -p 8000:8000 (or -P)

-p creates the NAT rule from host port to container port.

**Q170.** In Compose, `depends_on: [db]` is set, yet the api crashes because Postgres isn't accepting connections yet. What is the fix?

**Answer:** Add a healthcheck to db and use depends_on with condition: service_healthy

depends_on alone only controls start order, not readiness.

**Q171.** You run `docker compose down` on a stack with a named volume pgdata. What happens to the database data?

**Answer:** It's kept: down removes containers and networks but keeps named volumes (unless -v is given)

Named volumes outlive containers.

**Q172.** A .env file next to docker-compose.yml has APP_PORT=8080 and the service uses `ports: ["${APP_PORT}:8000"]`. Is APP_PORT available as an environment variable inside the container?

**Answer:** No: .env substitutes values into the compose file; add environment: - APP_PORT=${APP_PORT} to pass it through

The three mechanisms differ: ENV in the image, environment:/-e at runtime, and .env for compose-file substitution.

**Q173.** Which setup stops the proxy container from reaching the database?

**Answer:** Put db on an internal network shared only with the api, and the proxy on a public network only

Services need a shared network to talk. Verified by parsing a compose file: proxy-&gt;db False, api-&gt;db True.

**Q174.** Which image reference means an official Docker Hub image?

**Answer:** nginx:1.27 (shorthand for docker.io/library/nginx:1.27)

No namespace means library/ on Docker Hub.

**Q175.** Why is `myapp:latest` a poor tag to deploy in production?

**Answer:** It's mutable and can silently point at a different image; pin to a git SHA or version

Immutable tags make deployments reproducible and traceable.

**Q176.** You want the smallest, hardest-to-abuse final image for a Node API. Which Dockerfile ending is best?

```dockerfile
FROM gcr.io/distroless/nodejs20-debian12
COPY --from=build /app/dist ./dist
USER nonroot
CMD ["dist/server.js"]
```

**Answer:** This one: distroless has no shell or package manager, runs as non-root, and uses a build stage artifact

A shell-less image gives an attacker nothing to pivot with.

**Q177.** Which run flags make the root filesystem immutable while still allowing /tmp?

**Answer:** --read-only --tmpfs /tmp

--tmpfs gives in-memory writable paths without weakening the read-only guarantee.

**Q178.** How do you give a Swarm service a database password without an environment variable value?

**Answer:** Create a Docker secret and read it from /run/secrets/&lt;name&gt;, e.g. POSTGRES_PASSWORD_FILE

Secrets are mounted in memory as files, never in the writable layer.

**Q179.** A new image version must replace the old one with no dropped requests. Which combination is right?

**Answer:** Health-gated rolling update (start-first, rollback on failure) plus graceful shutdown

Never remove capacity before its replacement is provably ready.

**Q180.** A volume holds production data. Which backup is correct?

**Answer:** A throwaway container mounting the volume read-only and writing a tar to a host directory, plus a tested restore into a new volume

Test restores; a backup never restored is a hope, not a plan.

**Q181.** Containers on the default bridge can't reach each other by name, but they can on a network created with `docker network create app-net`. Why?

**Answer:** User-defined bridge networks provide embedded DNS name resolution

The legacy default bridge only supports IPs and --link.

**Q182.** In CI you push images tagged with the commit SHA. What does that buy you?

**Answer:** Traceability: every image maps to the exact source that built it

It's essential when debugging production weeks later.

**Q183.** When is Kubernetes justified over Compose or Swarm?

**Answer:** Many hosts with varied hardware, multiple teams needing RBAC and namespaces, metric-based autoscaling, or the Helm/operator ecosystem

For a single team with a handful of hosts and predictable load, Swarm or Compose is often the right size.

## Practical coding questions

### Isolate the tiers

Fill in the Compose file so that the proxy can reach the api, the api can reach the db, but the proxy cannot reach the db; only the proxy publishes a port; and the data network is internal. The checker parses your YAML and prints the results. (Needs PyYAML: pip install pyyaml.)

Starter code:

```
import yaml
COMPOSE = """
services:
  proxy:
    image: nginx:1.25
    # TODO: ports and networks
  api:
    build: ./api
    # TODO: networks
  db:
    image: postgres:16
    volumes: ["pgdata:/var/lib/postgresql/data"]
    # TODO: networks
# TODO: declare the networks (data must be internal)
volumes:
  pgdata:
"""
cfg = yaml.safe_load(COMPOSE)
svc = cfg["services"]
def can_reach(a, b):
    return bool(set(svc[a].get("networks", [])) & set(svc[b].get("networks", [])))
for a, b in [("proxy", "db"), ("api", "db"), ("proxy", "api")]:
    print(f"{a}->{b}: {can_reach(a, b)}")
print("published:", sorted(n for n, s in svc.items() if s.get("ports")))
print("internal networks:", sorted(n for n, v in cfg["networks"].items() if v and v.get("internal")))
```

Expected output:

```text
proxy->db: False
api->db: True
proxy->api: True
published: ['proxy']
internal networks: ['data']
```

??? success "Show a solution"

    ```
    import yaml
    COMPOSE = """
    services:
      proxy:
        image: nginx:1.25
        ports: ["8080:80"]
        networks: [public]
      api:
        build: ./api
        networks: [public, data]
      db:
        image: postgres:16
        volumes: ["pgdata:/var/lib/postgresql/data"]
        networks: [data]
    networks:
      public:
      data:
        internal: true
    volumes:
      pgdata:
    """
    cfg = yaml.safe_load(COMPOSE)
    svc = cfg["services"]
    def can_reach(a, b):
        return bool(set(svc[a].get("networks", [])) & set(svc[b].get("networks", [])))
    for a, b in [("proxy", "db"), ("api", "db"), ("proxy", "api")]:
        print(f"{a}->{b}: {can_reach(a, b)}")
    print("published:", sorted(n for n, s in svc.items() if s.get("ports")))
    print("internal networks:", sorted(n for n, v in cfg["networks"].items() if v and v.get("internal")))
    ```

    Services can talk only if they share a network. proxy is on public, db on data, and api on both, so api bridges them. internal: true stops the data network from reaching outside. Verified by running the checker (PyYAML) against the solution.

### Reorder for the build cache

The NAIVE Dockerfile reinstalls dependencies on every code edit. Write GOOD so that editing app.py rebuilds only COPY . ., while editing requirements.txt rebuilds the install too. The simulation applies the rule 'once one layer misses the cache, every later layer misses'.

Starter code:

```
NAIVE = """FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install -r requirements.txt
CMD ["python", "app.py"]"""

GOOD = """FROM python:3.12-slim
# TODO: reorder so dependencies are installed before the source is copied
"""

def rebuilt(dockerfile, changed):
    """Layers that must rebuild: once one layer misses the cache, every later layer misses too."""
    missed, out = False, []
    for line in dockerfile.splitlines():
        op, _, rest = line.partition(" ")
        if op not in ("FROM", "WORKDIR", "COPY", "RUN"):
            continue  # CMD only adds metadata
        if op == "COPY":
            src = rest.split()[0]
            if src == "." or src in changed:
                missed = missed or bool(src == "." and changed) or src in changed
        if missed:
            out.append(line)
    return out

print("naive, edit app.py ->", rebuilt(NAIVE, {"app.py"}))
print("good, edit app.py ->", rebuilt(GOOD, {"app.py"}))
print("good, edit requirements.txt ->", rebuilt(GOOD, {"requirements.txt"}))
```

Expected output:

```text
naive, edit app.py -> ['COPY . .', 'RUN pip install -r requirements.txt']
good, edit app.py -> ['COPY . .']
good, edit requirements.txt -> ['COPY requirements.txt .', 'RUN pip install -r requirements.txt', 'COPY . .']
```

??? success "Show a solution"

    ```
    NAIVE = """FROM python:3.12-slim
    WORKDIR /app
    COPY . .
    RUN pip install -r requirements.txt
    CMD ["python", "app.py"]"""

    GOOD = """FROM python:3.12-slim
    WORKDIR /app
    COPY requirements.txt .
    RUN pip install -r requirements.txt
    COPY . .
    CMD ["python", "app.py"]"""

    def rebuilt(dockerfile, changed):
        """Layers that must rebuild: once one layer misses the cache, every later layer misses too."""
        missed, out = False, []
        for line in dockerfile.splitlines():
            op, _, rest = line.partition(" ")
            if op not in ("FROM", "WORKDIR", "COPY", "RUN"):
                continue  # CMD only adds metadata
            if op == "COPY":
                src = rest.split()[0]
                if src == "." or src in changed:
                    missed = missed or bool(src == "." and changed) or src in changed
            if missed:
                out.append(line)
        return out

    print("naive, edit app.py ->", rebuilt(NAIVE, {"app.py"}))
    print("good, edit app.py ->", rebuilt(GOOD, {"app.py"}))
    print("good, edit requirements.txt ->", rebuilt(GOOD, {"requirements.txt"}))
    ```

    Copy the manifest and install dependencies first, then copy the source. Source edits then only invalidate the last COPY. Verified by running the simulation against the solution.

---

More free practice: the [Docker Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
