---
description: "Private Registries — Docker Hub is a public registry; most organizations also need a private registry for proprietary images — either self-hosted or a…"
---

# 05 · Private Registries

Docker Hub is a public registry; most organizations also need a
**private registry** for proprietary images — either self-hosted or a
managed offering (ECR, GCR, Azure ACR, GitHub Container Registry, Harbor).
This module covers running the open-source `registry` image yourself and
authenticating against any registry.

## Running a self-hosted registry

```bash
docker run -d --name registry -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  registry:2
```

This starts a minimal, unauthenticated registry listening on port 5000.
It's immediately usable for pushing/pulling on the host it runs on, with
images persisted in the `registry-data` volume across restarts.

```bash
docker tag myapp:1.0 localhost:5000/myapp:1.0
docker push localhost:5000/myapp:1.0
docker pull localhost:5000/myapp:1.0
```

The registry hostname/port becomes part of the image name — this is how
Docker's client decides where to push/pull, distinguishing
`localhost:5000/myapp` from Docker Hub's implicit `docker.io/library/myapp`.

## Adding authentication

An unauthenticated registry is fine for local experimentation only.
Basic auth via `htpasswd`:

```bash
mkdir auth
docker run --rm --entrypoint htpasswd httpd:2.4 -Bbn deploy-user supersecret > auth/htpasswd

docker run -d --name registry -p 5000:5000 \
  -v registry-data:/var/lib/registry \
  -v "$(pwd)/auth:/auth" \
  -e REGISTRY_AUTH=htpasswd \
  -e REGISTRY_AUTH_HTPASSWD_REALM="Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  registry:2
```

```bash
docker login localhost:5000
# Username: deploy-user
# Password: supersecret
```

## TLS: required for anything beyond `localhost`

Docker refuses to push/pull against a non-`localhost` registry over
plain HTTP by default (treating it as "insecure") unless explicitly told
otherwise. For a real deployment, terminate TLS in front of the registry
(a reverse proxy, or the registry's own TLS config) with a real or
internal CA certificate. For local testing only, a registry can be added
to the Engine's explicit insecure-registry allowlist:

```json
// /etc/docker/daemon.json
{
  "insecure-registries": ["registry.internal.example.com:5000"]
}
```

This is a deliberate, host-wide opt-out of transport security and should
never be used for a registry reachable outside a fully trusted network.

## Authenticating against a managed cloud registry

```bash
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 123456789.dkr.ecr.us-east-1.amazonaws.com

docker tag myapp:1.0 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
docker push 123456789.dkr.ecr.us-east-1.amazonaws.com/myapp:1.0
```

`--password-stdin` avoids the password appearing in shell history or
`ps` output, unlike an inline `--password` flag — the credential is
piped in directly rather than passed as a process argument.

## Worked example: CI pushing to a private registry

```bash
echo "$REGISTRY_PASSWORD" | docker login registry.example.com -u ci-bot --password-stdin
docker build -t registry.example.com/team/myapp:${CI_COMMIT_SHA} .
docker push registry.example.com/team/myapp:${CI_COMMIT_SHA}
docker logout registry.example.com
```

`docker logout` at the end of a CI job removes the cached credential from
the build agent's config, so it isn't left behind for a later,
potentially less-trusted job on shared infrastructure.

## How It Actually Works

**Where `docker login` actually stores the credential.** By default,
`docker login` writes the registry hostname and a base64-encoded
`username:password` pair into `~/.docker/config.json` under an `auths`
key — plaintext-equivalent (base64 is encoding, not encryption) unless a
credential helper is configured. On systems with one available, Docker
instead delegates to the OS's native secret store — `docker-credential-osxkeychain`
on macOS, `docker-credential-wincred` on Windows, `pass` or `secretservice`
on Linux — storing only a reference to the helper in `config.json` and
the actual secret in that OS-managed, encrypted store. This is why
copying `~/.docker/config.json` between machines "just to reuse
credentials" doesn't work in the credential-helper case: the file only
contains a pointer, and the real secret lives in a keychain scoped to
the original machine/user.

**Why the registry hostname is baked into the image reference itself,
not passed as a separate flag.** Every image reference Docker parses has
an implicit or explicit registry component:
`[registry[:port]/]repository[:tag]`. When no registry is given, the
client defaults to `docker.io` (and, for a single-segment name like
`nginx`, further defaults to the `library/` namespace). Push/pull
operations extract that registry component and look up matching
credentials from `config.json`'s `auths` map keyed by exact hostname —
which is precisely why `docker tag myapp:1.0 localhost:5000/myapp:1.0`
is a *required* step before pushing to a non-default registry: without
that registry-qualified tag, `docker push myapp:1.0` would target
Docker Hub, not your private registry, regardless of which registry
you're currently logged into.

## Exercise

Run a local unauthenticated registry on port 5000, push a small image to
it, then stop and remove the registry *container* (but not its volume),
start a fresh registry container reusing the same volume, and confirm
the previously pushed image can still be pulled — demonstrating that a
registry's actual storage is decoupled from any one container instance,
the same volume/container-lifecycle separation covered in Level 2,
module 07.
