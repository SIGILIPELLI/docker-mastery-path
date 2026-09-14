# 03 · Environment Variables & .env Files

Containers are configured mostly through environment variables rather
than config files baked into the image, so the same image can run
differently in dev, staging, and production without being rebuilt.

## Setting variables at `docker run` time

```bash
docker run -e DATABASE_URL=postgres://db:5432/app -e DEBUG=false myapp
```

```bash
# From a file instead of individual -e flags
docker run --env-file .env.production myapp
```

An `--env-file` is a plain `KEY=VALUE` list, one per line, no quoting of
values needed for simple strings:

```dotenv
DATABASE_URL=postgres://db:5432/app
DEBUG=false
LOG_LEVEL=info
```

## `.env` files with Compose

Compose treats a file literally named `.env` in the project directory
specially: it's loaded automatically and its values are available for
**variable substitution inside `docker-compose.yml` itself**, not just
injected into containers.

```dotenv
# .env
POSTGRES_VERSION=16
APP_PORT=8000
```

```yaml
services:
  db:
    image: postgres:${POSTGRES_VERSION}
  web:
    build: .
    ports:
      - "${APP_PORT}:8000"
```

`docker compose config` renders the final YAML with substitutions applied
— use it to verify what's actually being sent to the Engine before
trusting a nested `${VAR}` expression.

## Distinguishing three separate mechanisms

| Mechanism | Where it applies | Purpose |
|---|---|---|
| `ENV` in Dockerfile | Baked into the image | Defaults that ship with the image itself |
| `environment:` / `-e` | Container runtime | Per-run/per-environment overrides |
| `.env` (Compose only) | Compose YAML parsing | Substituting values *into* the compose file text, before it's even sent to the Engine |

A common mistake is expecting `.env` to inject variables into the
container the way `environment:` does — it doesn't, unless you also
reference `${VAR}` under `environment:` explicitly:

```yaml
services:
  web:
    environment:
      - APP_PORT=${APP_PORT}   # explicit passthrough into the container
```

## Worked example: environment-specific overrides

```bash
docker compose --env-file .env.staging up -d
```

Overriding which `.env` file Compose reads lets one `docker-compose.yml`
serve multiple environments, provided the file only contains values that
differ (ports, image tags, feature flags) rather than secrets — secrets
belong in Docker secrets or a secret manager (Level 3, module 09), not in
`.env` files that tend to get committed by accident.

## How It Actually Works

**Precedence order for a variable defined in more than one place.**
Docker resolves overlapping definitions in a fixed order, highest
priority last-write-wins: Dockerfile `ENV` sets the baseline baked into
the image; `docker compose run -e` or `docker run -e` on the command line
overrides it at container-create time; a `.env`-substituted value inside
`environment:` in the compose file sits below an explicit shell-exported
variable of the same name if you also export it before running
`docker compose up` (shell environment wins over `.env` file for
substitution purposes). Concretely, `docker inspect` on the resulting
container shows only the final, flattened list under `Config.Env` — there
is no runtime trace of which layer a given value came from, which is why
`docker compose config` (rendering pre-Engine) and `docker inspect`
(post-Engine) are the two tools you actually use to debug a
"wrong value" mystery, each answering a different half of the question.

**Why environment variables, not files, are the primary configuration
channel for containers.** A container image is meant to be immutable and
shared across environments (the same digest promoted from staging to
production); baking per-environment config into the filesystem would mean
rebuilding the image per environment, defeating that guarantee. Unix
processes have always had `environ` as a lightweight, universally
supported key-value channel passed at `exec()` time — Docker just exposes
it as a first-class run-time input (stored as part of the container's
config, injected into `environ` before the container's PID 1 process is
`exec()`'d) so the same image digest can be parameterized differently in
each place it runs, with the twelve-factor-app convention treating env
vars as the canonical place for anything that varies between deploys.

## Exercise

Create a `.env` file with `IMAGE_TAG=1.25` and a `docker-compose.yml`
whose `nginx` service uses `image: nginx:${IMAGE_TAG}`. Run
`docker compose config` to confirm the substitution happened, then
override it for one run without editing either file:
`IMAGE_TAG=1.24 docker compose config` — confirm the shell-exported value
wins over the `.env` file's value.
