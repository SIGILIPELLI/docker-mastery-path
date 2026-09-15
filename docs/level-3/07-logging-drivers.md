---
description: "Logging Drivers & Centralized Logging — Module 05 of Level 2 relied on the default json-file logging driver. For production, especially with many hosts…"
---

# 07 · Logging Drivers & Centralized Logging

Module 05 of Level 2 relied on the default `json-file` logging driver.
For production, especially with many hosts and short-lived containers,
logs need to leave the host entirely and land in a central aggregation
system before the container (and its log file) disappears.

## Setting a logging driver per container

```bash
docker run -d --log-driver=syslog \
  --log-opt syslog-address=udp://logs.example.com:514 \
  myapp
```

```bash
docker run -d --log-driver=json-file \
  --log-opt max-size=10m --log-opt max-file=3 \
  myapp
```

Even sticking with `json-file`, `max-size`/`max-file` cap total disk
usage per container by rotating — without them, a chatty container's log
file grows unbounded, the exact disk-filling risk flagged back in Level
2, module 05.

## Common drivers

| Driver | Destination |
|---|---|
| `json-file` (default) | Local JSON-lines file, readable via `docker logs` |
| `local` | A more space-efficient local binary format, still readable via `docker logs` |
| `syslog` | A syslog daemon (local or remote) |
| `journald` | The systemd journal |
| `fluentd` | A Fluentd collector, typically forwarding onward to Elasticsearch/Loki/etc. |
| `awslogs` | Amazon CloudWatch Logs directly |
| `gelf` | Graylog/Logstash via the GELF protocol |
| `none` | Discard entirely — no log storage at all |

A driver other than `json-file`/`local`/`journald` generally means
`docker logs` **stops working** for that container — the daemon is
shipping output straight to the external system instead of also keeping
a locally readable copy, so `docker logs` on a `fluentd`-configured
container typically returns nothing useful; you'd check the aggregator
instead.

## Setting a daemon-wide default

```json
// /etc/docker/daemon.json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Setting sane rotation defaults daemon-wide (rather than remembering
`--log-opt` on every `docker run`) is the single most common fix for
"disk filled up with container logs" incidents.

## Using a log-shipping sidecar instead of a driver

An alternative to configuring the driver directly is the sidecar pattern
from module 04 — a Fluent Bit (or Filebeat, Vector, etc.) container reads
the same `json-file` logs Docker already produces (mounting
`/var/lib/docker/containers` read-only) and ships them onward, giving you
`docker logs` locally *and* centralized shipping simultaneously, at the
cost of one more container to run and secure.

```yaml
services:
  app:
    build: .
  log-shipper:
    image: fluent/fluent-bit:latest
    volumes:
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
```

## Setting a logging driver in Compose

```yaml
services:
  api:
    build: .
    logging:
      driver: "gelf"
      options:
        gelf-address: "udp://logs.example.com:12201"
        tag: "{{.Name}}"
```

## Worked example: comparing the two approaches operationally

Direct driver (`awslogs`, `gelf`, `syslog`, etc.): simplest to configure,
but loses local `docker logs` access and ties the container's log
pipeline tightly to the daemon-level driver — changing where logs go
means changing every container's config or the daemon default and
restarting affected containers. Sidecar shipping json-file logs: keeps
`docker logs` working locally, decouples the shipping pipeline (you can
change/upgrade the shipper independently), but requires the shipper to
have read access to `/var/lib/docker/containers` and to correctly parse
Docker's json-file format including log rotation boundaries.

## How It Actually Works

**Where a logging driver actually sits in the log's path.** As
established in Level 2 module 05, the daemon holds the read end of the
pipe connected to a container's stdout/stderr from the moment it starts.
The logging driver is the code that runs *on that read end*: `json-file`
writes each line to a local file; `syslog`/`gelf`/`fluentd` instead
format and forward each line over the network to the configured
destination, using that protocol's own framing (RFC 5424 for syslog, the
GELF JSON+chunking format for GELF, Fluentd's forward protocol for
fluentd) — in every case the daemon itself is the one reading and
relaying, not a separate agent watching a file. This is exactly why
switching drivers on an already-running container isn't possible without
recreating it: `--log-driver` is fixed at container-create time as part
of how the daemon wires up that pipe's consumer.

**Why sidecar-based shipping instead reads a file, and what "tailing a
rotating file correctly" actually requires.** With `json-file` still
selected as the driver, log lines physically exist as JSON objects (one
per line) in `/var/lib/docker/containers/<id>/<id>-json.log`, rotated by
renaming the current file and starting a fresh one once `max-size` is
hit. A naive `tail -f` breaks across a rotation (the file descriptor it
holds now points at the renamed, no-longer-growing file). Log shippers
built for this instead track files by inode, not by path, and
periodically re-check whether the path they're tailing still refers to
the same inode — detecting a rotation by an inode/path mismatch rather
than by any signal from Docker itself, since Docker's `json-file` driver
does not notify external processes about rotation at all.

## Exercise

Set `max-size: "1m"` and `max-file: "2"` on a container that logs
continuously and verbosely (`docker run -d --log-opt max-size=1m
--log-opt max-file=2 alpine sh -c "while true; do echo $(date); sleep
0.01; done"`). Watch `/var/lib/docker/containers/<id>/` on the host (root
access required) and confirm you see rotation happen — never more than 2
files, each capped near 1MB — while `docker logs` for the container keeps
working across the rotation.
