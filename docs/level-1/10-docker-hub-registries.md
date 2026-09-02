# 10 · Docker Hub & Registries

Images need somewhere to live so they can be shared between machines —
your laptop, a CI runner, a production server. That's what a **registry**
is for, and **Docker Hub** is the default, most widely used one.

## What a registry is

A registry stores images, organized into **repositories** (by name), each
holding multiple **tags** (versions). When you run `docker pull` or
`docker run` with an image Docker doesn't have locally, it contacts a
registry (Docker Hub, unless configured otherwise) to fetch it.

```bash
docker pull nginx:1.27
```

Behind the scenes this:

1. Resolves `nginx` to the full reference `docker.io/library/nginx:1.27`
   (Docker Hub is the implicit default registry, and `library/` is the
   implicit namespace for Docker's official, curated images).
2. Downloads the image's manifest (a list of the layers that make it up).
3. Downloads any layers not already present locally, in parallel.
4. Assembles them into a usable local image.

## Official images vs user/organization images

| Reference | Meaning |
|---|---|
| `nginx` | Shorthand for `docker.io/library/nginx` — an **official image**, maintained/reviewed by Docker Hub |
| `bitnami/nginx` | An image published under the `bitnami` user/org namespace |
| `myusername/myapp` | An image you publish under your own Docker Hub username |
| `ghcr.io/someorg/someimage` | An image hosted on GitHub Container Registry instead of Docker Hub |

Official images (no namespace prefix) are a good default choice for base
images — they're maintained, regularly updated for security patches, and
documented — but plenty of legitimate images live under a
user/organization namespace, and you'll publish your own project's images
there too.

## Creating an account and logging in

Docker Hub accounts are created at hub.docker.com. Once you have one:

```bash
docker login
# Username: yourusername
# Password: ********
# Login Succeeded
```

`docker login` stores a credential (by default in `~/.docker/config.json`,
ideally backed by your OS credential store rather than plaintext) that
subsequent `push`/`pull` commands use to authenticate.

## Tagging an image for your namespace

Docker Hub requires images you push to be named
`<your-username>/<repo-name>[:tag]`. If you built an image locally as
`hello-app:1.0`, you retag it (without rebuilding) before pushing:

```bash
docker build -t hello-app:1.0 .
docker tag hello-app:1.0 yourusername/hello-app:1.0
```

`docker tag` doesn't copy any data — it just adds another name pointing
at the same image ID, which you can confirm with `docker images` showing
both tags sharing one `IMAGE ID`.

## Pushing and pulling

```bash
docker push yourusername/hello-app:1.0
```

```
The push refers to repository [docker.io/yourusername/hello-app]
5f70bf18a086: Pushed
a1b2c3d4e5f6: Pushed
1.0: digest: sha256:9c3d... size: 1573
```

Once pushed, anyone (for a public repository) can pull and run it:

```bash
docker pull yourusername/hello-app:1.0
docker run yourusername/hello-app:1.0
```

Docker Hub repositories default to **public**; free accounts get a
limited number of private repositories, and paid plans expand that limit
— check current Docker Hub pricing for specifics, since these limits
change over time.

## Rate limits and anonymous pulls

Docker Hub applies pull rate limits to anonymous and free-tier accounts
(the exact numbers have changed over the platform's history, so check
Docker Hub's current documentation rather than relying on a specific
number here). If you hit `toomanyrequests: You have reached your pull
rate limit`, `docker login` with an authenticated account typically
raises the limit, since authenticated pulls get a higher allowance than
anonymous ones.

## Other registries

Docker Hub isn't the only option, and the `docker` CLI works with any
registry implementing the standard registry API:

```bash
# GitHub Container Registry
docker login ghcr.io
docker tag hello-app:1.0 ghcr.io/yourusername/hello-app:1.0
docker push ghcr.io/yourusername/hello-app:1.0

# A self-hosted private registry
docker tag hello-app:1.0 registry.internal.example.com/hello-app:1.0
docker push registry.internal.example.com/hello-app:1.0
```

Whenever an image reference includes a hostname before the first `/`
(and that hostname looks like a domain, e.g. contains a dot or a port),
Docker treats it as pointing at that specific registry instead of Docker
Hub. Level 3 covers running your own private registry in detail.

## Searching Docker Hub

```bash
docker search postgres
# NAME                 DESCRIPTION                       STARS   OFFICIAL
# postgres             The PostgreSQL object-relational…  15000   [OK]
# bitnami/postgresql   Bitnami container image for Post…  400
```

`docker search` queries Docker Hub directly from the CLI, though many
people find browsing hub.docker.com in a browser more convenient for
reading full descriptions and tag lists.

## Worked example: publish, then pull on a "different machine"

```bash
docker build -t counter-app:1.0 .
docker tag counter-app:1.0 yourusername/counter-app:1.0
docker login
docker push yourusername/counter-app:1.0

# Simulate a fresh machine by removing the local image
docker rmi yourusername/counter-app:1.0 counter-app:1.0

docker run --rm yourusername/counter-app:1.0
# Docker pulls it fresh from Docker Hub and runs it, proving the
# published image is self-contained and independent of your local build
```

## Exercise

Create a free Docker Hub account if you don't have one, `docker login`
from the CLI, then tag and push any small image you've built in this
level (e.g. the Dockerfile from module 05/06) under
`yourusername/<some-name>:1.0`. Remove the local image with `docker
rmi`, then `docker pull` it back down and run it, confirming the round
trip through the registry worked end to end.
